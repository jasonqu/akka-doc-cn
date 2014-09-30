# 容错

如[Actor系统](../chapter2/02_actor_systems.md)中所述，每一个actor是其子actor的监管者，而且每一个actor会定义一个处理错误的监管策略。这个策略制定以后就不能修改，因为它被集成为actor系统结构必须的一部分。

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

我选择了一些非常著名的异常类型来演示[监管与监控](../chapter2/04_supervision_and_monitoring.md)中描述的错误处理方式。首先，它是一对一策略，意思是每一个子actor会被单独处理（多对一的策略与之相似，唯一的区别在于任何决策都应用于监管者的所有子actor，而不仅仅是出错的那一个）。这里我们对重启的频率作了限制，即每分钟最多进行10次重启；所有这样的设置都可以被空着，也就是说，相应的限制并不被采用, 留下了设置重启频率的绝对上限或让重启无限进行的可能性。如果超出上限则子actor将被终止。

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








































