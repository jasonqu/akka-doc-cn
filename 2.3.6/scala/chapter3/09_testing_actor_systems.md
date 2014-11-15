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













