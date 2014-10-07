# 有限状态机(FSM)

###概述
FSM (有限状态机) 可以mixin到akka Actor中，其概念在[Erlang 设计原则](http://www.erlang.org/documentation/doc-4.8.2/doc/design_principles/fsm.html)中有最好的描述。

一个 FSM 可以描述成一组具有如下形式的关系 :

  **State(S) x Event(E) -> Actions (A), State(S')**

这些关系的意思可以这样理解：

  *如果我们当前处于状态S，发生了E事件，则我们应执行操作A，然后将状态转换为S’。

###一个简单的例子
为了演示`FSM` trait的大部分功能，考虑一个actor，它接收到一组突然爆发的消息而将其送入邮箱队列，然后在消息爆发期过后或收到flush请求时再对消息进行发送。

首先，假设以下所有代码都使用这些import语句：

```scala
import akka.actor.{ Actor, ActorRef, FSM }
import scala.concurrent.duration._
```

我们的 “Buncher” actor的契约是接收或产生以下消息：

```scala
// received events
case class SetTarget(ref: ActorRef)
case class Queue(obj: Any)
case object Flush
 
// sent events
case class Batch(obj: immutable.Seq[Any])
```

``SetTarget``用来启动，为``Batches``设置发送目标；``Queue``会添加到内部队列而``Flush``标志着消息爆发的结束。

```scala
// states
sealed trait State
case object Idle extends State
case object Active extends State
 
sealed trait Data
case object Uninitialized extends Data
case class Todo(target: ActorRef, queue: immutable.Seq[Any]) extends Data
```

这个actor可以处于两种状态: 队列中没有消息（即``Idle``）或有消息（即``Active``）。只要一直有消息进来并且没有flush请求，它就停留在active状态。这个actor的内部状态数据是由批消息的发送目标actor引用和实际的消息队列组成.

现在让我们看看我们的FSM actor的框架:

```scala
class Buncher extends Actor with FSM[State, Data] {
 
  startWith(Idle, Uninitialized)
 
  when(Idle) {
    case Event(SetTarget(ref), Uninitialized) =>
      stay using Todo(ref, Vector.empty)
  }
 
  // transition elided ...
 
  when(Active, stateTimeout = 1 second) {
    case Event(Flush | StateTimeout, t: Todo) =>
      goto(Idle) using t.copy(queue = Vector.empty)
  }
 
  // unhandled elided ...
 
  initialize()
}
```

基本策略就是声明actor类, 混入`FSM` trait，并将可能的状态和数据设定为类型参数。在actor的内部使用一个DSL来声明状态机：

* `startsWith`定义初始状态和初始数据
* 然后对每一个状态有一个`when(<state>) { ... }`声明待处理的事件（可以是多个，传入的`PartialFunction`将用`orElse`进行连接）
* 最后使用`initialize`来启动它，这会执行到初始状态的转换并启动定时器（如果需要的话）。

在这个例子中，我们从``Idle``和``Uninitialized``状态开始, 这两种状态下只处理``SetTarget()``消息；``stay``准备结束这个事件的处理而不离开当前状态，而``using``使得FSM将其内部状态（这时为``Uninitialized``）替换为一个新的包含目标actor引用的``Todo()``对象。``Active``状态声明了一个状态超时，意思是如果1秒内没有收到消息，将生成一个``FSM.StateTimeout``消息。在本例中这与收到``Flush``指令消息具有相同的效果，即转回``Idle``状态并将内部队列重置为空vector。但消息是如何进入队列的？由于在两种状态下都要做这件事， 我们利用了任何``when()``块未处理的消息被发送到``whenUnhandled()``块这个事实：

```scala
whenUnhandled {
  // common code for both states
  case Event(Queue(obj), t @ Todo(_, v)) =>
    goto(Active) using t.copy(queue = v :+ obj)
 
  case Event(e, s) =>
    log.warning("received unhandled request {} in state {}/{}", e, stateName, s)
    stay
}
```

这里第一个case是将``Queue()``请求加入内部队列中并进入``Active``状态（当然显然地，如果已经在``Active``状态则停留），前提是收到``Queue()``时FSM数据不是``Uninitialized``。否则——及其它所有未命中的情况——第二个case记录一个警告到日志并保持内部状态。

最后剩下的只有``Batches``实际上是如何发送到目标的，这里我们使用``onTransition``机制：你可以声明多个这样的块，在状态切换发生时（即只有当状态真正改变时）所有的块都将被尝试来作匹配。

```scala
onTransition {
  case Active -> Idle =>
    stateData match {
      case Todo(ref, queue) => ref ! Batch(queue)
    }
}
```

状态转换回调是一个偏函数，它以一对状态作为输入的——当前状态和下一个状态。FSM trait为此提供了一个方便的箭头形式的提取器，非常贴心地提醒你所匹配到的状态转换的方向。在状态转换过程中，如示例所示旧状态数据可以通过``stateData``获得，新状态数据可以通过``nextStateData``获得。

要确认这个buncher真实可用, 可以很简单地利用[测试Actor系统(Scala)](09_testing_actor_systems.md)中的工具写一个测试，它方便地将ScalaTest trait融入``AkkaSpec``中：

```scala
import akka.actor.Props
import scala.collection.immutable
 
class FSMDocSpec extends MyFavoriteTestFrameWorkPlusAkkaTestKit {
 
  // fsm code elided ...
 
  "simple finite state machine" must {
 
    "demonstrate NullFunction" in {
      class A extends Actor with FSM[Int, Null] {
        val SomeState = 0
        when(SomeState)(FSM.NullFunction)
      }
    }
 
    "batch correctly" in {
      val buncher = system.actorOf(Props(classOf[Buncher], this))
      buncher ! SetTarget(testActor)
      buncher ! Queue(42)
      buncher ! Queue(43)
      expectMsg(Batch(immutable.Seq(42, 43)))
      buncher ! Queue(44)
      buncher ! Flush
      buncher ! Queue(45)
      expectMsg(Batch(immutable.Seq(44)))
      expectMsg(Batch(immutable.Seq(45)))
    }
 
    "not batch if uninitialized" in {
      val buncher = system.actorOf(Props(classOf[Buncher], this))
      buncher ! Queue(42)
      expectNoMsg
    }
  }
}
```

###参考
#####FSM Trait 及 FSM Object
`FSM` trait只能被混入到`Actor`子类中。这里选择了 使用self类型的写法而不是继承`Actor`，这样是为了标明事实上创建的是一个actor。

```scala
class Buncher extends Actor with FSM[State, Data] {
 
  // fsm body ...
 
  initialize()
}
```



TODO
FSM trait 有两个类型参数 :

所有状态名称的父类型，通常是一个sealed trait，状态名称作为case object来继承它
状态数据的类型，由 FSM 模块自己跟踪.
注意
状态数据与状态名称一起描述了状态机的内部状态; 如果你坚持这种方案，不向FSM类中加入可变量成员，你就可以充分享受在 一些周知的位置改变所有内部状态的好处.





. 如果你希望直接访问象StateTimeout这样的符号，建议导入 FSM object 的所有成员. 通常这个导入写在状态机的定义中:




















































































































































































