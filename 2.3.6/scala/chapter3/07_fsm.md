# 有限状态机(FSM)

###概述
FSM (有限状态机) 可以mixin到akka Actor中，其概念在[Erlang 设计原则](http://www.erlang.org/documentation/doc-4.8.2/doc/design_principles/fsm.html)中有最好的描述。

一个 FSM 可以描述成一组具有如下形式的关系 :

  **State(S) x Event(E) -> Actions (A), State(S')**

这些关系的意思可以这样理解：

> 如果我们当前处于状态S，发生了E事件，则我们应执行操作A，然后将状态转换为S’。

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

基本策略就是声明actor类，混入`FSM` trait，并将可能的状态和数据设定为类型参数。在actor的内部使用一个DSL来声明状态机：

* `startsWith`定义初始状态和初始数据
* 然后对每一个状态有一个`when(<state>) { ... }`声明待处理的事件（可以是多个，传入的`PartialFunction`将用`orElse`进行连接）
* 最后使用`initialize`来启动它，这会执行到初始状态的转换并启动定时器（如果需要的话）。

在这个例子中，我们从``Idle``和``Uninitialized``状态开始，这两种状态下只处理``SetTarget()``消息；``stay``准备结束这个事件的处理而不离开当前状态，而``using``使得FSM将其内部状态（这时为``Uninitialized``）替换为一个新的包含目标actor引用的``Todo()``对象。``Active``状态声明了一个状态超时，意思是如果1秒内没有收到消息，将生成一个``FSM.StateTimeout``消息。在本例中这与收到``Flush``指令消息具有相同的效果，即转回``Idle``状态并将内部队列重置为空vector。但消息是如何进入队列的？由于在两种状态下都要做这件事，我们利用了任何``when()``块未处理的消息被发送到``whenUnhandled()``块这个事实：

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

要确认这个buncher真实可用，可以很简单地利用[测试Actor系统(Scala)](09_testing_actor_systems.md)中的工具写一个测试，它方便地将ScalaTest trait融入``AkkaSpec``中：

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
`FSM` trait只能被混入到`Actor`子类中。这里选择了 使用self类型的写法而不是继承`Actor`，这样是为了标明事实上创建的是一个actor。//TODO这里代码有误？

```scala
class Buncher extends Actor with FSM[State, Data] {
 
  // fsm body ...
 
  initialize()
}
```

> 注意

> FSM特质定义``receive``方法处理内部消息，并将其它一切通过FSM逻辑传递（根据当前状态）。当重写``receive``方法，请牢记例如超时状态处理取决于实际通过FSM逻辑传递的消息。

`FSM` trait 有两个类型参数 :
1. 所有状态名称的父类型，通常是一个sealed trait，状态名称作为case object来继承它
2. 状态数据的类型，由`FSM`模块自己跟踪。

<a name="fsm-philosophy"></a>
> 注意

> 状态数据与状态名称一起描述了状态机的内部状态；如果你坚持这种模式，不向FSM类中加入可变量成员，你就可以充分享受在一些周知的位置改变所有内部状态的好处.

#####定义状态
状态的定义是通过一次或多次调用

	when(<name>[, stateTimeout = <timeout>])(stateFunction)

方法。

给定的名称对象必须与为`FSM` trait指定的第一个参数类型相匹配。这个对象将被用作一个hash表的键，所以你必须确保它正确地实现了`equals` `hashCode`方法；特别是它不能为可变变量。满足这些条件的最简单的就是 case objects。

如果给定了`stateTimeout`参数，那么所有到这个状态的转换，包括停留，缺省都会收到这个超时。初始化转换时显式指定一个超时可以用来覆盖这个缺省行为，更多信息见[发起状态转换](#initiating-transitions)。在操作执行的过程中可以通过使用`setStateTimeout(state, duration)`方法来修改任何状态的超时时间。这使得运行时配置（例如通过外部消息）成为可能。

参数`stateFunction`是一个`PartialFunction[Event, State]`，可以很方便地用偏函数的语法来指定，见下例：

```scala
when(Idle) {
  case Event(SetTarget(ref), Uninitialized) =>
    stay using Todo(ref, Vector.empty)
}
 
when(Active, stateTimeout = 1 second) {
  case Event(Flush | StateTimeout, t: Todo) =>
    goto(Idle) using t.copy(queue = Vector.empty)
}
```

该`Event(msg: Any, data: D)` case类将FSM持有的数据类型参数化，从而可以方便地进行模式匹配。

> 警告

> 它要求你为所有可能的FSM状态定义处理程序，否则尝试切换到未声明的状态时会有失败。

推荐的实践做法是，将状态声明为对象，继承自一个sealed特质，然后验证是否对每一个状态都有``when``子句。如果你想要一些状态“不被处理”（详见下文），它仍然需要像这样声明：

```scala
when(SomeState)(FSM.NullFunction)
```
#####定义初始状态

每个FSM都需要一个起点，用以下代码声明

```scala
startWith(state, data[, timeout])
```

其中可选的超时参数将覆盖所有为期望的初始状态所指定的值。如果你想要取消缺省的超时，使用`Duration.Inf`。

#####未处理事件
如果一个状态未能处理一个收到的事件，日志中将记录一条警告。这种情况下如果你想做点其它的事，你可以使用`whenUnhandled(stateFunction)`来指定：

```scala
whenUnhandled {
  case Event(x: X, data) =>
    log.info("Received unhandled event: " + x)
    stay
  case Event(msg, _) =>
    log.warning("Received unknown event: " + msg)
    goto(Error)
}
```

在此处理程序中可以使用`stateName`方法查询FSM的状态。

**重要**：这个处理器不会入栈叠加，这意味着`whenUnhandled`的每一次调用都会覆盖先前指定的处理程序。

###<a name="initiating-transitions"></a>发起状态转换
任何`stateFunction`的结果都必须是下一个状态的定义，除非是终止FSM，这种情况在[从内部终止](#termination-from-inside)中介绍。状态定义可以是当前状态，由`stay`指令描述，或由`goto(state)`指定的另一个状态。结果对象可以通过下面列出的修饰器作进一步的限制：

* `forMax(duration)`

  这个修饰器为新状态指定了一个状态超时。这意味着将启动一个定时器，它过期时将向FSM发送一个`StateTimeout`消息。其间接收到任何其它消息，定时器都将被取消；你可以确定的事实是`StateTimeout`消息不会在任何一个中间消息之后被处理。

  这个修饰器也可以用于覆盖任何对目标状态指定的缺省超时。如果要取消缺省超时，可以使用`Duration.Inf`。

* `using(data)`

  这个修饰器用给定的新数据替换旧的状态数据。如果你遵循[上面的建议](#fsm-philosophy)，这是内部状态数据被修改的唯一位置。

* `replying(msg)`

  这个修饰器为当前处理完的消息发送一个应答，不同的是它不会改变状态转换。

所有的修饰器都可以链式调用来获得优美简洁的表达方式：

```scala
when(SomeState) {
  case Event(msg, _) =>
    goto(Processing) using (newData) forMax (5 seconds) replying (WillDo)
}
```

事实上这里所有的括号都不是必须的，但它们在视觉上将修饰器和它们的参数区分开，因而使代码对于他人来说有更好的可读性。

> 注意

> 请注意``return``语句不可以用于`when`或类似的代码块中；这是Scala的限制。要么使用``if () ... else ...``重构你的代码，要么将它改写到一个方法定义中。

#####监控状态转换
在概念上，转换发生在“两个状态之间”，也就是在你放在事件处理代码块执行的任何操作之后；这是显然的，因为只有在事件处理逻辑返回了值以后，才能确定下一个状态。相对于设置内部状态变量，你不需要担心操作顺序的细节，因为FSM actor中的所有代码都是在一个线程中运行的。

######内部监控
到目前为止，FSM DSL都围绕着状态和事件。另外一种视角是将其描述成一系列的状态转换。是通过这个方法实现的

    onTransition(handler)

它将操作与状态转换联系起来，而不是联系状态与事件。这个处理器是一个偏函数，它以一对状态作为输入；不需要结果状态，因为不可能改变正在进行的状态转换。

```scala
onTransition {
  case Idle -> Active => setTimer("timeout", Tick, 1 second, true)
  case Active -> _    => cancelTimer("timeout")
  case x -> Idle      => log.info("entering Idle from " + x)
}
```

提取器`->`用来解开状态对，并以清晰的形式表达了状态转换的方向。与通常的模式匹配一样，可以用`_`来表示不关心的内容；或者你可以将不关心的状态绑定到一个无约束的变量，例如像上一个例子那样供记日志使用。

也可以向`onTransition`传递一个以两个状态为参数的函数，此时你的状态转换处理逻辑是定义成方法的：

```scala
onTransition(handler _)
 
def handler(from: StateType, to: StateType) {
  // handle it here ...
}
```

用这个方法注册的处理器是堆栈迭加的，这样你可以将`onTransition`块和`when`块分散定义以适应设计的需要。但需要注意的是，*所有的处理器对每一次状态转换都会被调用*，而不只是最先匹配的那个。这是有意设计的，使得你可以将某一部分状态转换处理放在某一个地方，而不用担心先前的定义会屏蔽后面的逻辑；当然这些操作还是按定义的顺序执行的。

> 注意

> 这种内部监控可以用于通过状态转换来构建你的FSM，这样在添加新的目标状态时，不会忘记例如在离开某个状态时，取消定时器这种操作。

######外部监控
可以通过发送一个`SubscribeTransitionCallBack(actorRef)`消息注册外部actor，来接收状态转换的通知。这个被命名的actor将立即收到`CurrentState(self, stateName)`消息，并在之后每次进入新状态时收到`Transition(actorRef, oldState, newState)`消息。可以通过向FSM actor发送`UnsubscribeTransitionCallBack(actorRef)`来注销外部监控actor。

停止一个监听器，而不注销，将不会从注册列表中移除它；需要在停止监听器前使用`UnsubscribeTransitionCallback`。

#####转换状态
给``when()``传递的偏函数参数，可以使用Scala充分的函数式编程工具来转换。为了保留类型推断，还有一个辅助函数，它可以在通用处理逻辑中使用，并被应用到不同子句：

```scala
when(SomeState)(transform {
  case Event(bytes: ByteString, read) => stay using (read + bytes.length)
} using {
  case s @ FSM.State(state, read, timeout, stopReason, replies) if read > 1000 =>
    goto(Processing)
})
```

不用说此方法的参数也可能被存储，被多次使用，例如在几个不同的``when()``代码块中应用相同的转换：

```scala
val processingTrigger: PartialFunction[State, State] = {
  case s @ FSM.State(state, read, timeout, stopReason, replies) if read > 1000 =>
    goto(Processing)
}
 
when(SomeState)(transform {
  case Event(bytes: ByteString, read) => stay using (read + bytes.length)
} using processingTrigger)
```

#####定时器
除了状态超时，FSM还管理以`String`类型名称为标识的定时器。你可以用下面代码设置定时器

    setTimer(name, msg, interval, repeat)


其中`msg`是经过`interval`时间以后发送的消息对象。 如果`repeat`设成`true`，定时器将以`interval`参数指定的时间段重复规划。添加新的计时器之前，具有相同名称的任何现有计时器将被自动取消。


可以用下面代码取消定时器

    cancelTimer(name)

取消操作确保立即执行，这意味着在这个调用之后，定时器已经规划的消息将不会执行，即使定时器已经发起并入队该消息。任何定时器的状态可以用下面的代码进行查询

    isTimerActive(name)

这些具名定时器是对状态超时的补充，因为它们不受中间收到的其它消息的影响。

#####<a name="termination-from-inside"></a>从内部终止
可以像下面这样指定结果状态来终止FSM

    stop([reason[, data]])

其中的`reason`必须是`Normal`（默认值）、`Shutdown`或`Failure(reason)`之一，可以提供第二个参数来改变状态数据，在终止处理器中可以使用该数据。

> 注意

> 必须注意`stop`并不会中止当前的操作并立即停止FSM。`stop`操作必须像状态转换一样从事件处理器中返回（但要注意``return``语句不能用在`when`代码块中）。

```scala
when(Error) {
  case Event("stop", _) =>
    // do cleanup ...
    stop()
}
```

你可以用`onTermination(handler)`来指定当FSM停止时要运行的代码。其中的handler是以一个`StopEvent(reason, stateName, stateData)`为参数的偏函数：

```scala
onTermination {
  case StopEvent(FSM.Normal, state, data)         => // ...
  case StopEvent(FSM.Shutdown, state, data)       => // ...
  case StopEvent(FSM.Failure(cause), state, data) => // ...
}
```

对于使用`whenUnhandled`的场合，这个处理器不会堆栈迭加，所以每次`onTermination`调用都会替换先前指定的处理器。

#####从外部终止
当与FSM关联的`ActorRef`被`stop`方法停止后，它的`postStop` hook 将被执行。在`FSM`特质中的缺省实现是执行`onTermination`处理器（如果有的话）来处理`StopEvent(Shutdown, ...)`事件。

> 警告

> 如果你重写`postStop`并而希望你的`onTermination `处理器被调用，不要忘记调用``super.postStop``。

###测试和调试有限状态机
在开发和调试过程中，FSM和其它actor一样需要照顾。[测试有限状态机](09_testing_actor_systems.md#testfsmref)以及下文中介绍了一些专门的工具。

#####事件跟踪
[配置文件](../chapter2/09_configuration.md)中的``akka.actor.debug.fsm``打开用`LoggingFSM`实例完成的事件跟踪日志：

```scala
import akka.actor.LoggingFSM
class MyFSM extends Actor with LoggingFSM[StateType, Data] {
  // body elided ...
}
```

这个FSM将以DEBUG级别记录日志：

  * 所有处理完的事件，包括`StateTimeout`和计划的定时器消息
  * 所有具名定时器的设置和取消
  * 所有的状态转换

生命周期变化及特殊消息可以如[Actor](09_testing_actor_systems.md#actor-logging-scala)中所述进行日志记录。

#####滚动事件日志
`LoggingFSM`特质为FSM添加了一个新的特性：一个滚动的事件日志，它可以在debugging中使用（跟踪为什么FSM会进入某个失败的状态）或其它的新用法：

```scala
import akka.actor.LoggingFSM
class MyFSM extends Actor with LoggingFSM[StateType, Data] {
  override def logDepth = 12
  onTermination {
    case StopEvent(FSM.Failure(_), state, data) =>
      val lastEvents = getLog.mkString("\n\t")
      log.warning("Failure in state " + state + " with data " + data + "\n" +
        "Events leading up to this point:\n\t" + lastEvents)
  }
  // ...
}
```

`logDepth`缺省值为0，意思是关闭事件日志。

> 警告

> 日志缓冲区是在actor创建时分配的，这也是为什么logDepth的配置使用了虚方法调用。如果你想用一个``val``对其进行覆盖，必须保证它的初始化在`LoggingFSM`的初始化之前完成，而且在缓冲区分配完成后不要修改`logDepth`返回的值。

事件日志的内容可以用`getLog`方法获取，它返回一个`IndexedSeq[LogEntry]`，其中最老的条目下标为0。

###示例
在[Typesafe Activator](http://www.typesafe.com/platform/getstarted)的模板工程[Akka FSM in Scala](http://www.typesafe.com/activator/template/akka-sample-fsm-scala)中，可以找到一个比Actor’s `become/unbecome`更大的FSM示例。


