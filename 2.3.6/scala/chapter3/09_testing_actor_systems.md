# 测试Actor系统

* [TestKit 实例 (Scala)](testkit-example.md)

对于任何软件开发，自动化测试都是开发过程中一个重要组成部分。actor 模型对于代码单元如何划分，它们之间如何交互提供了一种新的视角，这对如何编写测试也造成了影响.

Akka 有一个专门的模块——`akka-testkit`来支持不同层次上的测试, 测试很明显有两个类别:

* 测试独立的、不包括actor模型的代码，即没有多线程的内容；这意味着给事件发生的次序给定，有完全确定的行为，没有任何并发考虑，这在下文中称为 **单元测试（Unit Testing）**。
* 测试（多个）包装过的actor，包括多线程调度；这意味着事件的次序没有确定性，但由于使用了actor模型而不需要考虑并发，这在下文中被称为 **集成测试（Integration Testing）**。

当然这两个类型有着不同的测试粒度, 单元测试通常是白盒测试，而集成测试是对完整的actor网络进行的功能测试。是否把并发考虑为测试的一部分，是其中重要的区别。我们提供的工具将在下面的章节中详细介绍。

> 注意

> 请确保在依赖项中加入`akka-testkit`模块。

###用`TestActorRef`做同步单元测试
`Actor`类中的业务逻辑测试可以分为两部分: 首先，每个原子操作必须独立工作，然后输入的事件序列必须被正确处理，即使事件的次序存在一些可能的变化。前者是单线程单元测试的主要使用场景，而后者可以在集成测试中进行确认。

通常，`ActorRef`将实际的`Actor`实例与外界隔离开，唯一的通信通道是actor的邮箱。这个限制是单元测试的障碍，所以我们引入了`TestActorRef`。这个特殊类型的引用是专门为测试而设计的，它允许以两种方式访问actor：通过获取实际actor实例的引用，或者通过调用或查询actor的行为（`receive`）。下文中对每一种方式都有专门的章节介绍。

#####获取一个`Actor`的引用

能够访问到实际的`Actor`对象使得所有的传统单元测试技术都可以用于测试其中的方法。可以像这样获取一个引用：

```scala
import akka.testkit.TestActorRef
 
val actorRef = TestActorRef[MyActor]
val actor = actorRef.underlyingActor
```

由于`TestActorRef`是actor类型的高阶类型（泛型），因此它返回的底层actor具有正确的静态类型。这样以后你就可以象平常一样将任何单元测试工具用于actor上了。

#####测试有限状态机

如果你要测试的actor是一个`FSM`，你可以使用专门的`TestFSMRef`，它拥有普通`TestActorRef`的所有功能，并且额外地允许访问其内部状态：

```scala
import akka.testkit.TestFSMRef
import akka.actor.FSM
import scala.concurrent.duration._
 
val fsm = TestFSMRef(new TestFsmActor)
 
val mustBeTypedProperly: TestActorRef[TestFsmActor] = fsm
 
assert(fsm.stateName == 1)
assert(fsm.stateData == "")
fsm ! "go" // being a TestActorRef, this runs also on the CallingThreadDispatcher
assert(fsm.stateName == 2)
assert(fsm.stateData == "go")
 
fsm.setState(stateName = 1)
assert(fsm.stateName == 1)
 
assert(fsm.isTimerActive("test") == false)
fsm.setTimer("test", 12, 10 millis, true)
assert(fsm.isTimerActive("test") == true)
fsm.cancelTimer("test")
assert(fsm.isTimerActive("test") == false)
```

由于Scala类型推测的限制，只有一个如上所示的工厂方法，所以你很可能需要写像``TestFSMRef(new MyFSM)``这样的代码，而不是想象中的类似`ActorRef`的``TestFSMRef[MyFSM]``。上例所示的所有方法都直接访问FSM的状态，不作任何同步；这在使用`CallingThreadDispatcher`且没有其它线程参与的情况下是合适的，但如果你实际上需要 处理定时器事件可能会导致意外的情形，因为它们是在`Scheduler`线程中执行的。

#####测试Actor的行为
当消息派发器调用actor的逻辑来处理一条消息时，它实际上是对当前注册到actor的行为调用了`apply`。行为的初始值是代码中声明的`receive`方法的返回值，但可以通过对外部消息的响应调用`become`和`unbecome`来改变这个行为。所有这些特性使得进行actor的行为测试不太容易。因此`TestActorRef`提供了一种不同的操作方式来对`Actor`的测试进行补充：它支持所有正常的`ActorRef`中的操作。发往actor的消息在当前线程中同步处理，应答象往常一样回送。这个技巧来自下面所介绍的[`CallingThreadDispatcher`](#CallingThreadDispatcher)；这个派发器被隐式地用于所有实例化为`TestActorRef`的actor。

```scala
import akka.testkit.TestActorRef
import scala.concurrent.duration._
import scala.concurrent.Await
import akka.pattern.ask
 
val actorRef = TestActorRef(new MyActor)
// hypothetical message stimulating a '42' answer
val future = actorRef ? Say42
val Success(result: Int) = future.value.get
result should be(42)
```

由于`TestActorRef`是`LocalActorRef`的子类，只是附加了一些特殊功能，所以像监管和重启这样的功能也能正常工作，但要注意：只要所有相关的actor都使用`CallingThreadDispatcher`，那么所有的执行过程都是严格同步的。一旦你增加了一些元素，其中包括比较复杂的定时任务，你就离开了单元测试的范畴，因为你必须要重新将异步性纳入考虑范围（在大多数情况下问题在于要等待希望的结果有机会发生）。

另一个在单线程测试中被覆盖的特殊点是`receiveTimeout`，由于包含了它会产生异步的`ReceiveTimeout`消息队列，因此与同步约定矛盾。

> 警告

> 综上所述：`TestActorRef`重写了两个成员: 它设置派发器为`CallingThreadDispatcher.global`，设置`receiveTimeout`为None。

#####介于两者之间方法：期望异常

如果你想要测试actor的行为，包括热替换，但是不涉及消息派发器，也不希望`TestActorRef`吞掉所有抛出的异常，那么为你准备了另一种模式: 只要使用 `TestActorRef` 的 `receive` 方法 , 这将会把消息转发给内部的 actor:

```scala
import akka.testkit.TestActorRef
 
val actorRef = TestActorRef(new Actor {
  def receive = {
    case "hello" => throw new IllegalArgumentException("boom")
  }
})
intercept[IllegalArgumentException] { actorRef.receive("hello") }
```

#####使用场景

当然你可以根据自己的测试需求来混合使用 `TestActorRef` 的不同用法:

* 一个常见的使用场景是在发送测试消息之前设置actor进入某个特定的内部状态
* 另一个场景是发送了测试消息之后验证内部状态转换的正确性

放心大胆地对各种可能性进行实验，如果你发现了有用的模式，快让Akka论坛知道它！常用操作甚至可能放进优美的DSL中，谁知道呢。

###用`TestKit`进行异步集成测试

当基本确定你的actor的业务逻辑是正确的之后，下一步就是确认它在目标环境中也能正确工作（如果actor个体都足够简单，可能是因为他们使用了 `FSM` 模块, 这也可以放到第一步). 关于环境的定义当然很大程度上由手头的问题和 你打算测试的程度来决定, 从功能/集成测试到完整的系统测试. 最简单的步骤包括测试过程（提供所期望的触发条件）、要测试的actor和接收应答的actor。大一些的系统将被测的actor替换成一组actor网络，将触发条件应用于不同的切入点，并整理将会从不同的输出位置发送的结果，但基本的原则保持不变，就是测试由一个单独的流程来驱动。

`TestKit` 类包含一组工具来简化这些常用的工作.

```scala
import akka.actor.ActorSystem
import akka.actor.Actor
import akka.actor.Props
import akka.testkit.{ TestActors, TestKit, ImplicitSender }
import org.scalatest.WordSpecLike
import org.scalatest.Matchers
import org.scalatest.BeforeAndAfterAll
 
class MySpec(_system: ActorSystem) extends TestKit(_system) with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
 
  def this() = this(ActorSystem("MySpec"))
 
  override def afterAll {
    TestKit.shutdownActorSystem(system)
  }
 
  "An Echo actor" must {
 
    "send back messages unchanged" in {
      val echo = system.actorOf(TestActors.echoActorProps)
      echo ! "hello world"
      expectMsg("hello world")
    }
 
  }
}
```

`TestKit` 中有一个名为 `testActor` 的actor作为将要被不同的 ``expectMsg...``断言检查的消息的入口，下面会详细介绍这些断言。 当混入了 `ImplicitSender` trait后, 这个测试actor会在整个测试过程中，被隐式地用作消息的发送者引用. `testActor` 也可以像平常一样被 发送给其它的actor，通常是订阅成为通知监听器. 有一套检查方法, 例如 接收所有匹配某些条件的消息，接收一系列固定的消息序列或类，在某段时间内收不到消息，等。

作为参数传递到TestKit构造函数的ActorSystem 可以通过`system`成员来访问。记得在测试完成后关闭actor系统 (即使是在测试失败的情况下) 以保证所有的actor——包括测试actor——被停止.


#####内置断言
上面提到的 `expectMsg `并不是唯一的对收到的消息进行断言的方法. 以下是完整的列表:

* `expectMsg[T](d: Duration, msg: T): T`
  给定的消息对象必须在指定的时间内到达；该消息对象将被返回.

* `expectMsgPF[T](d: Duration)(pf: PartialFunction[Any, T]): T`

  在给定的时间段内，必须有一条消息到达，必须为这类消息定义了偏函数；将收到的消息应用于偏函数并返回其结果。可以不指定时间段（这时需要一对空的括号）， 这时使用最深层的 [`within`](#testkit-within) 块中的期限.

* `expectMsgClass[T](d: Duration, c: Class[T]): T`

  在分配的时间片内必须接收到` Class `类型的对象实例；并返回该对象。 注意它的类型匹配是子类兼容的；如果需要类型是相等的，参考使用单个class参数的 `expectMsgAllClassOf` 。

* `expectMsgType[T: Manifest](d: Duration)`

  在分配的时间片内必须收到指定类型 (擦除后)的对象实例; 并返回该对象。这个方法基本上与 ``expectMsgClass(implicitly[ClassTag[T]].runtimeClass)``等价.

* `expectMsgAnyOf[T](d: Duration, obj: T*): T`
  在指定的时间内必须收到一个对象，而且此对象必须与传入的多个对象引用中至少一个相等( 用 ``==`` 进行比较); 并返回该对象.

* `expectMsgAnyClassOf[T](d: Duration, obj: Class[_ <: T]*): T`

  在指定的时间内必须收到一个对象，它必须指定的多个` Class` 中某一个类对象的实例; 并返回该对象.

* `expectMsgAllOf[T](d: Duration, obj: T*): Seq[T]`

  在指定时间内必须收到与指定的对象数组中相等数量的对象, 对每个收到的对象，必须至少有一个数组中的对象与之相等(用``==``进行比较) . 返回收到对象的完整序列.

* `expectMsgAllClassOf[T](d: Duration, c: Class[_ <: T]*): Seq[T]`

  在指定时间内必须收到与指定的 `Class` 数组相等数量的对象，对数组中的每一个类， 必须至少有一个对象的类与之相等(用 ``==``进行比较) (这 *不是* 子类兼容的类型检查). 返回收到对象的完整序列.

* `expectMsgAllConformingOf[T](d: Duration, c: Class[_ <: T]*): Seq[T]`

  在指定时间内必须收到与指定的 `Class` 数组相等数量的对象，对数组中的每个类必须至少有一个对象是这个类的实例。返回收到对象的完整序列.

* `expectNoMsg(d: Duration)`

  在指定时间内不能收到消息。如果在这个方法被调用之前已经收到了消息，并且没有用其它的方法将这些消息从队列中删除，这个断言也会失败。

* `receiveN(n: Int, d: Duration): Seq[AnyRef]`

  指定的时间内必须收到``n`` 条消息; 返回收到的消息.

* `fishForMessage(max: Duration, hint: String)(pf: PartialFunction[Any, Boolean]): Any`

  只要时间没有用完，而且偏函数匹配收到的消息并返回``false``，则一直接收消息. 返回使偏函数返回``true`` 的消息或抛出异常, 异常中会提供一些提示供debug使用。

除了消息接收断言，还有一些方法来对消息流提供帮助:

* `receiveOne(d: Duration): AnyRef`

  尝试等待给定的时间间隔来接收一个消息，如果失败则返回``null`` . 如果给定的 Duration 是0，则调用是非阻塞的（轮询模式）.
* `receiveWhile[T](max: Duration, idle: Duration, messages: Int)(pf: PartialFunction[Any, T]): Seq[T]`

  只要满足以下条件就收集消息：
  * 消息与偏函数匹配
  * 指定的时间还没用完
  * 在空闲的时间内收到了下一条消息
  * 消息数量还没有到上限

  返回收集到的所有消息. 时间上限缺省值是最深层的 [`within`](#testkit-within) 块中的剩余时间，空闲时间缺省为无限 (也就是禁止空闲超时功能). 期望的消息数量缺省值为 Int.MaxValue, 也就是不作这个限制.

* `awaitCond(p: => Boolean, max: Duration, interval: Duration)`

  每经过 `interval`时间就检查一下给定的条件，直到它返回 ``true`` 或者 ``max`` 时间用完了. 时间间隔缺省为 100 ms 而最大值缺省为最深层的 [`within`](#testkit-within)  块中的剩余时间.

  * `awaitAssert(a: => Any, max: Duration, interval: Duration)`

  在每个时间间隔验证一遍给定的断言函数，直到它不抛出异常，或``max``持续时间用完。如果在超时已到则抛出最后一个异常。间隔默认为 100 ms 和最大值默认为最深层的 [`within`](#testkit-within)  块中的剩余时间。
  TODO Akka文档中的重复

* `ignoreMsg(pf: PartialFunction[AnyRef, Boolean])`
  `ignoreNoMsg`

  内部的 `testActor` 包含一个偏函数用来忽略消息: 它只会将与偏函数不匹配或使函数返回``false``的消息放进队列。 这个函数可以用上面的方法进行设置和 重设; 每一次调用都会覆盖之前的函数，而不会迭加.

  这个功能在你想忽略一般的消息而只对指定的一些消息感兴趣时（例如测试日志系统时）比较有用。

#####期望日志消息
由于集成测试不允许进入参与测试的actor内部处理流程, 无法直接确认预料中的异常. 因此为了做这件事，只能使用日志系统：将普通的事件处理器替换成 `TestEventListener`并使用一个 `EventFilter` ，从而可以对日志消息，包括由于异常产生的日志，做断言:

```scala
import akka.testkit.EventFilter
import com.typesafe.config.ConfigFactory
 
implicit val system = ActorSystem("testsystem", ConfigFactory.parseString("""
  akka.loggers = ["akka.testkit.TestEventListener"]
  """))
try {
  val actor = system.actorOf(Props.empty)
  EventFilter[ActorKilledException](occurrences = 1) intercept {
    actor ! Kill
  }
} finally {
  shutdown(system)
}
```

如果出现的次数是具体的——如上所示——然后``intercept``将阻塞直到收到匹配消息达到这一数字，或``akka.test.filter-leeway``配置的超时时间用完了 （在传入的代码块返回后，时间开始计数）。超时时测试失败。

> 注意

> 一定要在你的 `application.conf`中 替换默认的日志记录器为`TestEventListener`，以启用此功能：

```
akka.loggers = [akka.testkit.TestEventListener]
```

<span id="testkit-within"></span>
#####对定时进行断言
功能测试的另一个重要部分与定时器有关：有些事件不能立即发生（如定时器）, 另外一些需要在时间期限内发生. 因此所有的进行检查的方法 都接收一个时间上限，不论是正面还是负面的结果都应该在这个时间之前获得。时间下限需要在这个检测方法之外进行检查，因而有一个新的工具来管理时间期限:

```scala
   within([min, ]max) {
     ...
   }
```

传给`within`的代码块必须在一个介于 `min` 和 `max`之间的`Duration`之前完成, 其中`min`缺省值为0. 将`max` 参数与块的启动时间相加得到的时间期限在所有检查方法块内部都可以隐式获得， 如果你没有指定`max`值，它会从最深层的 `within` 块继承这个值.

应注意如果代码块的最后一条接收消息断言是 `expectNoMsg` 或 `receiveWhile`, 对 `within` 的最终检查将被跳过，以避免由于唤醒延迟导致的错误的true值. 这意味着虽然其中每一个独立的断言仍然使用时间上限，整个代码块在这种情况下会有长度随机的延迟。

```scala
import akka.actor.Props
import scala.concurrent.duration._
 
val worker = system.actorOf(Props[Worker])
within(200 millis) {
  worker ! "some work"
  expectMsg("some result")
  expectNoMsg // will block for the rest of the 200ms
  Thread.sleep(300) // will NOT make this block fail
}
```

> 注意

> 所有的时间都以 ``System.nanoTime``为单位, 即它们描述的是墙上时间，而非CPU时间.

Ray Roestenburg 写了一篇关于使用 TestKit 的好文: [http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html]. 完整的示例也可以在[这里](09_1_testkit-example.md)找到.

#######考虑很慢的测试系统
你在跑得飞快的笔记本上使用的超时设置在高负载的Jenkins（或类似的）服务器上通常都会导致虚假的测试失败。 为了考虑这种情况，所有的时间上限都在内部乘以一个系数，这个系数来自 [配置文件](../chapter2/09_configuration.md)中的 ``akka.test.timefactor``, 缺省值为 1.

你也可以用``akka.testkit``包对象中的隐式转换来将同样的系数来作用于其它的时限，为 `Duration `添加扩展函数.

```scala
import scala.concurrent.duration._
import akka.testkit._
10.milliseconds.dilated
```

#####用隐式的ActorRef解决冲突

如果你希望在基于TestKit的测试中，消息发送者为 ``testActor`` 只需要在你的测试代码混入 ``ÌmplicitSender`` .

```scala
class MySpec(_system: ActorSystem) extends TestKit(_system) with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
```

#####使用多个探针 Actor
如果待测的actor会发送多个消息到不同的目标，在使用`TestKit`时可能会难以分辨到达 `testActor`的消息流. 另一种方法是用它来创建简单的探针actor，将它们插入到消息流中. 为了让这种方法更加强大和方便，我们提供了一个具体实现，称为 `TestProbe`. 它的功能可以用下面的小例子说明:

```scala
import scala.concurrent.duration._
import akka.actor._
import scala.concurrent.Future

class MyDoubleEcho extends Actor {
  var dest1: ActorRef = _
  var dest2: ActorRef = _
  def receive = {
    case (d1: ActorRef, d2: ActorRef) =>
      dest1 = d1
      dest2 = d2
    case x =>
      dest1 ! x
      dest2 ! x
  }
}

val probe1 = TestProbe()
val probe2 = TestProbe()
val actor = system.actorOf(Props[MyDoubleEcho])
actor ! ((probe1.ref, probe2.ref))
actor ! "hello"
probe1.expectMsg(500 millis, "hello")
probe2.expectMsg(500 millis, "hello")
```

这里我们用 `MyDoubleEcho`来模拟一个待测系统, 它会将输入镜像为两个输出. 关联两个测试探针来进行（最简单）行为的确认. 还有一个例子是两个相互协作的 actor A，B， A 发送消息给 B. 为了确认这个消息流，可以插入一个 `TestProbe` 作为A的目标, 使用转发功能或下文中的自动导向功能在测试上下文中包含一个真实的B.

还可以为探针配备自定义的断言来使测试代码更简洁清晰:
<<<<<<< HEAD
=======

```scala
ase class Update(id: Int, value: String)
 
val probe = new TestProbe(system) {
  def expectUpdate(x: Int) = {
    expectMsgPF() {
      case Update(id, _) if id == x => true
    }
    sender() ! "ACK"
  }
}

```

这里你拥有完全的灵活性，可以将`TestKit` 提供的工具与你自己的检测代码混合和匹配，并为它取一个有意义的名字。在实际开发中你的代码很可能比上面的示例要复杂；要充分利用工具的力量！
>>>>>>> refs/remotes/origin/gh-pages

```scala
ase class Update(id: Int, value: String)
 
val probe = new TestProbe(system) {
  def expectUpdate(x: Int) = {
    expectMsgPF() {
      case Update(id, _) if id == x => true
    }
    sender() ! "ACK"
  }
}

```

这里你拥有完全的灵活性，可以将`TestKit` 提供的工具与你自己的检测代码混合和匹配，并为它取一个有意义的名字。在实际开发中你的代码很可能比上面的示例要复杂；要充分利用工具的力量！

> 警告

> 任何从 ``TestProbe``发送到  另一个运行在 `CallingThreadDispatcher`上 的actor的消息，存在死锁的可能性，如果另一个actor也可能会向此探测器发送消息。`TestProbe.watch` 和 `TestProbe.unwatch `的实现也将向将空着发送消息，这意味着尝试监视它是危险的，例如从 `TestProbe`到`TestActorRef`  发送消息。

###从探针观察其它actor

`TestProbe `可以将自己注册为任意其他actor的DeathWatch：

```scala
val probe = TestProbe()
probe watch target
target ! PoisonPill
probe.expectTerminated(target)
```

#####对探针收到的消息进行应答
探针在可能的条件下，会记录通讯通道以便进行应答:

```scala
val probe = TestProbe()
val future = probe.ref ? "hello"
probe.expectMsg(0 millis, "hello") // TestActor runs on CallingThreadDispatcher
probe.reply("world")
assert(future.isCompleted && future.value == Some(Success("world")))
```

#####对探针收到的消息进行转发
假定一个象征性的actor网络中某目标 actor ``dest`` 从 actor ``source``收到一条消息. 如果你安排消息使其先发往 `TestProbe` ``probe`` , 你可以在保持网络功能的同时对消息流的容量和时限进行断言:

```scala
class Source(target: ActorRef) extends Actor {
  def receive = {
    case "start" => target ! "work"
  }
}
 
class Destination extends Actor {
  def receive = {
    case x => // Do something..
  }
}
val probe = TestProbe()
val source = system.actorOf(Props(classOf[Source], probe.ref))
val dest = system.actorOf(Props[Destination])
source ! "start"
probe.expectMsg("work")
probe.forward(dest)
```

目标`` dest`` actor 将先收到同样的消息，就象没有插入探针一样.

#####自动导向
将收到的消息放进队列以便以后处理，这种方法不错，但要保持测试运行并对其运行过程进行跟踪，你也可以为参与测试的探针(事实上是任何 `TestKit`)安装一个 `AutoPilot`（自动导向）。 自动导向在消息进入检查队列之前启动. 以下代码可以用来转发消息, 例如 ``A --> Probe --> B``, 只要满足一定的协约.

```scala
val probe = TestProbe()
probe.setAutoPilot(new TestActor.AutoPilot {
  def run(sender: ActorRef, msg: Any): Option[TestActor.AutoPilot] =
    msg match {
      case "stop" ⇒ None
      case x      ⇒ testActor.tell(x, sender); Some(this)
    }
})
```

###TODO
`run` 方法必须返回包含在Option 中的auto-pilot供下一条消息使用, 设置成 None 表示终止自动导向.

小心定时器断言
在使用测试探针时，within 块的行为可能会不那么直观：你需要记住 上文 所描述的期限仅对每一个探针的局部作用域有效。因此，探针 不会响应别的探针的期限，也不响应包含它的TestKit 实例的期限:

class SomeTest extends TestKit(_system: ActorSystem) with ImplicitSender {
 
  val probe = TestProbe()
 
  within(100 millis) {
    probe.expectMsg("hallo")  // 将永远挂起!
  }
}
这个测试将无限挂起，因为 expectMsg 调用看不到任何期限. 目前，使它正常工作的唯一方法是在代码中使用 probe.within ; 以后的版本可能会提供通过词法作用域内的隐式参数设置期限的方法.














































