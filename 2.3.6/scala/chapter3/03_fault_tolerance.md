# 容错

如[Actor系统](../chapter2/02_actor_systems.md)中所述，每一个actor都是其子actor的监管者，而且每一个actor会定义一个处理错误的监管策略。这个策略制定以后就不能修改，因为它被集成为actor系统结构所必须的一部分。

###错误处理实践
首先我们来看一个例子，演示处理数据库错误的一种方法，数据库错误是真实应用中的典型错误类型。当然在实际的应用中这要依赖于当数据库发生错误时能做些什么， 在这个例子中，我们使用尽量重新连接的方法。

阅读以下源码。其中的注释解释了错误处理的各个片段以及为什么要加上它们。我们还强烈建议运行这个例子，因为根据输出日志理解运行时发生的事情会比较容易。

* [容错示例图片](03-1_fault_tolerance_sample.md)
* [容错示例完整源代码](03-1_fault_tolerance_sample.md#full-source-code-of-the-fault-tolerance-sample)

###创建一个监管策略
以下章节更加深入地解释了错误处理机制和可选的方法。

为了演示我们假设有这样的策略:

```scala
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy._
import scala.concurrent.duration._
 
override val supervisorStrategy =
  OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
    case _: ArithmeticException      => Resume
    case _: NullPointerException     => Restart
    case _: IllegalArgumentException => Stop
    case _: Exception                => Escalate
  }
```

我选择了一些非常著名的异常类型来演示[监管与监控](../chapter2/04_supervision_and_monitoring.md)中描述的错误处理方式。首先，它是一对一策略，意思是每一个子actor会被单独处理（多对一的策略与之相似，唯一的区别在于任何决策都应用于监管者的所有子actor，而不仅仅是出错的那一个）。这里我们对重启的频率作了限制，即每分钟最多进行10次重启；所有这样的设置都可以被空着，也就是说，相应的限制并不被采用，留下了设置重启频率的绝对上限或让重启无限进行的可能性。如果超出上限则子actor将被终止。

构成主体的match语句的类型是``Decider``，它是一个``PartialFunction[Throwable, Directive]``。该部分将把子actor的失败类型映射到相应的指令上。

> 注意

> 如果在监管actor内部声明策略（而不是在伴生对象中），其决策者就能够以线程安全的方式访问actor的所有内部状态，包括获取对当前失败的子actor引用（作为失败消息的发送者）。

#####缺省的监管机制
如果定义的监管机制没有覆盖抛出的异常，将使用``Escalate``上溯机制。

如果某个actor没有定义监管机制，下列异常将被缺省地处理为：

* ``ActorInitializationException``将终止出错的子actor
* ``ActorKilledException``将终止出错的子actor
* ``Exception``将重启出错的子actor
* 其它的``Throwable``将被上溯传给父actor

如果异常一直被上溯到根监管者，在那儿也会用上述缺省方式进行处理。

你可以将自己的策略与默认策略结合：

```scala
import akka.actor.OneForOneStrategy
import akka.actor.SupervisorStrategy._
import scala.concurrent.duration._

override val supervisorStrategy =
  OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
    case _: ArithmeticException => Resume
    case t =>
      super.supervisorStrategy.decider.applyOrElse(t, (_: Any) => Escalate)
  }
```

#####停止监管策略
Erlang方式的策略是当actor失败时，终止它们，然后当DeathWatch通知子actor消失时，采取正确的措施。这一策略还提供了预打包为`SupervisorStrategy.stoppingStrategy`及伴随的`StoppingSupervisorStrategy`配置，当你想要使用``"/user"``监管者时可以使用它。

#####actor失败的日志记录
默认情况下``SupervisorStrategy``会日志记录失败，除非他们被上溯升级。升级的失败应该被树形结构中更高级的监管者处理，即可能在那里记录日志。

当实例化时，你可以通过将``loggingEnabled``设置为``false``来取消``SupervisorStrategy``的默认日志记录。自定义日志记录可以在``Decider``内完成。请注意当``SupervisorStrategy``是在监管actor内声明的时候，可以通过``sender``获取当前失败的子actor引用。

你也可以通过重写``logFailure``方法在你自己的``SupervisorStrategy``实现定制日志记录。

###监督顶级actor
顶级actor是指那些使用``system.actorOf()``创建的，而它们将是[User监管Actor](../chapter2/04_supervision_and_monitoring.md#user-guardian)的子actor。这里没有使用特殊规则，监管者只是应用了已配置的策略。

###测试应用
以下部分展示了实际中不同的指令的效果，为此我们需要创建一个测试环境。首先我们需要一个合适的监管者：

```scala
  import akka.actor.Actor
 
  class Supervisor extends Actor {
    import akka.actor.OneForOneStrategy
    import akka.actor.SupervisorStrategy._
    import scala.concurrent.duration._
 
    override val supervisorStrategy =
      OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
        case _: ArithmeticException      => Resume
        case _: NullPointerException     => Restart
        case _: IllegalArgumentException => Stop
        case _: Exception                => Escalate
      }
 
    def receive = {
      case p: Props => sender() ! context.actorOf(p)
    }
  }
```

该监管者将被用来创建一个可以做试验的子actor：

```scala
import akka.actor.Actor
 
class Child extends Actor {
  var state = 0
  def receive = {
    case ex: Exception => throw ex
    case x: Int        => state = x
    case "get"         => sender() ! state
  }
}
```

这个测试可以用[测试Actor系统(Scala)](09_testing_actor_systems.md)中的工具来进行简化, 比如``AkkaSpec``是``TestKit with WordSpec with
MustMatchers``的一个方便的混合

```scala
import akka.testkit.{ AkkaSpec, ImplicitSender, EventFilter }
import akka.actor.{ ActorRef, Props, Terminated }
 
class FaultHandlingDocSpec extends AkkaSpec with ImplicitSender {
 
  "A supervisor" must {
 
    "apply the chosen strategy for its child" in {
      // code here
    }
  }
}
```

现在我们来创建actor：

```scala
val supervisor = system.actorOf(Props[Supervisor], "supervisor")
 
supervisor ! Props[Child]
val child = expectMsgType[ActorRef] // 从 TestKit 的 testActor 中获取回应
```

第一个测试是为了演示``Resume``指令，我们试着将actor设为非初始状态然后让它出错：

```scala
child ! 42 // 将状态设为 42
child ! "get"
expectMsg(42)
 
child ! new ArithmeticException // crash it
child ! "get"
expectMsg(42)
```

可以看到错误处理指令完后仍能得到42的值。现在如果我们将错误换成更严重的``NullPointerException``，情况就不同了:

```scala
child ! new NullPointerException // crash it harder
child ! "get"
expectMsg(0)
```

而最后当致命的``IllegalArgumentException``发生时子actor将被其监管者终止：

```scala
watch(child) // have testActor watch “child”
child ! new IllegalArgumentException // break it
expectMsgPF() { case Terminated(`child`) => () }
```

到目前为止监管者完全没有被子actor的错误所影响，因为指令集确实处理了这些错误。而对于``Exception``，就不是这么回事了，监管者会将失败上溯传递。

```scala
supervisor ! Props[Child] // create new child
val child2 = expectMsgType[ActorRef]
 
watch(child2)
child2 ! "get" // verify it is alive
expectMsg(0)
 
child2 ! new Exception("CRASH") // escalate failure
expectMsgPF() {
  case t @ Terminated(`child2`) if t.existenceConfirmed => ()
}
```

监管者自己是被`ActorSystem`的顶级actor所监管的，顶级actor的缺省策略是对所有的``Exception``情况（注意``ActorInitializationException``和``ActorKilledException``例外）进行重启. 由于缺省的重启指令会杀死所有的子actor，所以我们知道（期望）可怜的子actor最终无法从这个失败中幸免。

如果这不是我们希望的行为（这取决于实际情况），我们需要使用一个不同的监管者来覆盖这个行为。

```scala
class Supervisor2 extends Actor {
  import akka.actor.OneForOneStrategy
  import akka.actor.SupervisorStrategy._
  import scala.concurrent.duration._
 
  override val supervisorStrategy =
    OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
      case _: ArithmeticException      => Resume
      case _: NullPointerException     => Restart
      case _: IllegalArgumentException => Stop
      case _: Exception                => Escalate
    }
 
  def receive = {
    case p: Props => sender() ! context.actorOf(p)
  }
  // override default to kill all children during restart
  override def preRestart(cause: Throwable, msg: Option[Any]) {}
}
```

在这个父actor之下，子actor在上溯的重启中得以幸免，在如下这个最后的测试中：

```scala
val supervisor2 = system.actorOf(Props[Supervisor2], "supervisor2")
 
supervisor2 ! Props[Child]
val child3 = expectMsgType[ActorRef]
 
child3 ! 23
child3 ! "get"
expectMsg(23)
 
child3 ! new Exception("CRASH")
child3 ! "get"
expectMsg(0)
```
