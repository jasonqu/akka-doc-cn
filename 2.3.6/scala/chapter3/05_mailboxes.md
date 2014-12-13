# 邮箱
一个Akka ``Mailbox``保存发往某个Actor的消息。通常每个Actor都拥有自己的邮箱，但也有例外，例如使用``BalancingPool``的所有路由子(routee)共享同一个邮箱实例。

###邮箱选择
#####为actor指定一个消息队列类型
为某个特定类型的actor指定一个特定类型的消息队列是有可能的，只要通过actor扩展`RequiresMessageQueue`参数化特质即可。下面是一个示例：

```scala
import akka.dispatch.RequiresMessageQueue
import akka.dispatch.BoundedMessageQueueSemantics
 
class MyBoundedActor extends MyActor
  with RequiresMessageQueue[BoundedMessageQueueSemantics]
```

`RequiresMessageQueue`特质的类型参数需要映射到配置中的邮箱，像这样：

```
bounded-mailbox {
  mailbox-type = "akka.dispatch.BoundedMailbox"
  mailbox-capacity = 1000
  mailbox-push-timeout-time = 10s
}
 
akka.actor.mailbox.requirements {
  "akka.dispatch.BoundedMessageQueueSemantics" = bounded-mailbox
}
```

现在每当你创建一个类型为`MyBoundedActor`的actor时，它会尝试得到一个有界的邮箱。如果actor在部署中具有一个不同的邮箱配置——直接地，或是通过一个指定的邮箱类型的调度器——那么它将覆盖此映射。

> 注意

> 为actor创建的邮箱队列类型，会和特质中要求的类型进行检查，如果队列没有实现要求的类型，则actor创建会失败。

#####为调度器指定一个消息队列类型
调度器也可能对actor使用的邮箱类型进行限制。一个例子是 BalancingDispatcher，它要求消息队列对多个并发的消费者是线程安全的。该约束是在配置的调度器一节中像下面这样设定：

```
  my-dispatcher {
    mailbox-requirement = org.example.MyInterface
  }
```

给定的约束是要求指定的类或者接口，必须是消息队列的实现的超类。如果出现冲突——例如如果actor要求的邮箱类型不能满足要求——则actor创建会失败。

#####如何选择邮箱类型
当一个actor创建时，`ActorRefProvider`首先确定将执行它的调度器。然后，邮箱确定如下：

1. 如果actor的部署配置节包含``mailbox``键，则其描述邮箱类型将被使用。
2. 如果actor的``Props``包含邮箱选择——即它调用了``withMailbox``——则其描述邮箱类型将被使用。
3. 如果调度器的配置节包含``mailbox-type``键，则该节内容将用于配置邮箱类型。
4. 如果该actor需要邮箱类型，如上文所述，然后该约束的映射将用于确定使用的邮箱类型；如果不能满足调度器的约束——如果有的话——将继续替换尝试。
5. 如果调度器需要一个邮箱类型，如上文所述，则该约束的映射将被用来确定要使用的邮箱类型。
6. 将使用默认邮箱``akka.actor.default-mailbox``。

#####默认邮箱
当未如上所述，指定使用邮箱时，将使用默认邮箱。默认情况它是无界的邮箱，由``java.util.concurrent.ConcurrentLinkedQueue``实现。

``SingleConsumerOnlyUnboundedMailbox``是一个更有效率的邮箱，而且它也可以用作默认邮箱，但它不能与 BalancingDispatcher一起使用。

``SingleConsumerOnlyUnboundedMailbox``作为默认邮箱的配置：

```
  akka.actor.default-mailbox {
    mailbox-type = "akka.dispatch.SingleConsumerOnlyUnboundedMailbox"
  }
```

#####哪项配置会传递给邮箱类型
每个邮箱类型由扩展`MailboxType`并带两个构造函数参数的类实现：一个`ActorSystem.Settings`对象和一个`Config`对象。后者通过actor系统配置的指定配置节被计算出来，重写其指定邮箱类型的配置路径的``id``键，并添加一个到默认邮箱配置节的回退。

###内置实现
Akka自带有一些内置的邮箱实现：

* UnboundedMailbox
  * 默认邮箱
  * 底层是一个``java.util.concurrent.ConcurrentLinkedQueue``
  * 阻塞: 否
  * 有界: 否
  * 配置名称："unbounded" 或 "akka.dispatch.UnboundedMailbox"

* SingleConsumerOnlyUnboundedMailbox
  * 底层是一个非常高效的多生产者单消费者队列，不能被用于BalancingDispatcher
  * 阻塞: 否
  * 有界: 否
  * 配置名称："akka.dispatch.SingleConsumerOnlyUnboundedMailbox"

* BoundedMailbox
  * 底层是一个``java.util.concurrent.LinkedBlockingQueue``
  * 阻塞: 是
  * 有界: 是
  * 配置名称："bounded" 或 "akka.dispatch.BoundedMailbox"

* UnboundedPriorityMailbox
  * 底层是一个``java.util.concurrent.PriorityBlockingQueue``
  * 阻塞: 是
  * 有界: 否
  * 配置名称："akka.dispatch.UnboundedPriorityMailbox"

* BoundedPriorityMailbox
  * 底层是一个 ``java.util.PriorityBlockingQueue``包装为``akka.util.BoundedBlockingQueue``
  * 阻塞: 是
  * 有界: 是
  * 配置名称："akka.dispatch.BoundedPriorityMailbox"

###邮箱配置示例
如何创建一个PriorityMailbox：

```scala
import akka.dispatch.PriorityGenerator
import akka.dispatch.UnboundedPriorityMailbox
import com.typesafe.config.Config
 
// We inherit, in this case, from UnboundedPriorityMailbox
// and seed it with the priority generator
class MyPrioMailbox(settings: ActorSystem.Settings, config: Config)
  extends UnboundedPriorityMailbox(
    // Create a new PriorityGenerator, lower prio means more important
    PriorityGenerator {
      // 'highpriority messages should be treated first if possible
      case 'highpriority => 0
 
      // 'lowpriority messages should be treated last if possible
      case 'lowpriority  => 2
 
      // PoisonPill when no other left
      case PoisonPill    => 3
 
      // We default to 1, which is in between high and low
      case otherwise     => 1
    })
```

并添加到配置文件:

```
prio-dispatcher {
  mailbox-type = "docs.dispatcher.DispatcherDocSpec$MyPrioMailbox"
  //Other dispatcher configuration goes here
}
```

以下示例演示如何使用它:

```scala
// We create a new Actor that just prints out what it processes
class Logger extends Actor {
  val log: LoggingAdapter = Logging(context.system, this)
 
  self ! 'lowpriority
  self ! 'lowpriority
  self ! 'highpriority
  self ! 'pigdog
  self ! 'pigdog2
  self ! 'pigdog3
  self ! 'highpriority
  self ! PoisonPill
 
  def receive = {
    case x => log.info(x.toString)
  }
}
val a = system.actorOf(Props(classOf[Logger], this).withDispatcher(
  "prio-dispatcher"))
 
/*
 * Logs:
 * 'highpriority
 * 'highpriority
 * 'pigdog
 * 'pigdog2
 * 'pigdog3
 * 'lowpriority
 * 'lowpriority
 */
```

也可以像这样直接配置邮箱类型：

```
prio-mailbox {
  mailbox-type = "docs.dispatcher.DispatcherDocSpec$MyPrioMailbox"
  //Other mailbox configuration goes here
}
 
akka.actor.deployment {
  /priomailboxactor {
    mailbox = prio-mailbox
  }
}
```

然后可以在部署环境像这样使用它：

```scala
import akka.actor.Props
val myActor = context.actorOf(Props[MyActor], "priomailboxactor")
```

或像这样在代码中：

```scala
import akka.actor.Props
val myActor = context.actorOf(Props[MyActor].withMailbox("prio-mailbox"))
```

###创建自己的邮箱类型
例子比文字更有说服力：

```scala
import akka.actor.ActorRef
import akka.actor.ActorSystem
import akka.dispatch.Envelope
import akka.dispatch.MailboxType
import akka.dispatch.MessageQueue
import akka.dispatch.ProducesMessageQueue
import com.typesafe.config.Config
import java.util.concurrent.ConcurrentLinkedQueue
import scala.Option
 
// Marker trait used for mailbox requirements mapping
trait MyUnboundedMessageQueueSemantics
 
object MyUnboundedMailbox {
  // This is the MessageQueue implementation
  class MyMessageQueue extends MessageQueue
    with MyUnboundedMessageQueueSemantics {
 
    private final val queue = new ConcurrentLinkedQueue[Envelope]()
 
    // these should be implemented; queue used as example
    def enqueue(receiver: ActorRef, handle: Envelope): Unit =
      queue.offer(handle)
    def dequeue(): Envelope = queue.poll()
    def numberOfMessages: Int = queue.size
    def hasMessages: Boolean = !queue.isEmpty
    def cleanUp(owner: ActorRef, deadLetters: MessageQueue) {
      while (hasMessages) {
        deadLetters.enqueue(owner, dequeue())
      }
    }
  }
}
 
// This is the Mailbox implementation
class MyUnboundedMailbox extends MailboxType
  with ProducesMessageQueue[MyUnboundedMailbox.MyMessageQueue] {
 
  import MyUnboundedMailbox._
 
  // This constructor signature must exist, it will be called by Akka
  def this(settings: ActorSystem.Settings, config: Config) = {
    // put your initialization code here
    this()
  }
 
  // The create method is called to create the MessageQueue
  final override def create(owner: Option[ActorRef],
                            system: Option[ActorSystem]): MessageQueue =
    new MyMessageQueue()
}
```

然后在派发器配置中，或邮箱配置中以你定义的邮箱类型的全称作为“mailbox-type”的值。

> 注意

> 一定要定义一个以``akka.actor.ActorSystem.Settings``和``com.typesafe.config.Config``为参数的构造函数，该构造函数将以反射的方式被调用来创建你的邮箱类型。第二个传入的配置参数是配置文件中使用这个邮箱类型的派发器或邮箱的描述；该邮箱类型会为每一个使用它的派发器或邮箱创建一个实例。

此外可以作为派发器的约束这样使用邮箱：

```
custom-dispatcher {
  mailbox-requirement =
  "docs.dispatcher.MyUnboundedJMessageQueueSemantics"
}

akka.actor.mailbox.requirements {
  "docs.dispatcher.MyUnboundedJMessageQueueSemantics" =
  custom-dispatcher-mailbox
}

custom-dispatcher-mailbox {
  mailbox-type = "docs.dispatcher.MyUnboundedJMailbox"
}
```

或通过在你的actor类上定义约束，像这样：

```scala
class MySpecialActor extends Actor
  with RequiresMessageQueue[MyUnboundedMessageQueueSemantics] {
  // ...
}
```

###``system.actorOf``的特殊语义
为了使``system.actorOf``同步和非阻塞，并且返回类型保持为`ActorRef`（并且保持返回的ref是完全函数式的语义），这种情况下会进行特殊处理。在幕后，一种空心的actor引用会被构造，然后发送到系统的监管者，该监管者实际创建actor和它的上下文，并把它们传到引用内部。直到这些发生以后，发送到该`ActorRef`的消息会被本地排队，而只有当填入真实信息后，它们才会被传入真实的邮箱，因此，

```scala
val props: Props = ...
// this actor uses MyCustomMailbox, which is assumed to be a singleton
system.actorOf(props.withDispatcher("myCustomMailbox")) ! "bang"
assert(MyCustomMailbox.instance.getLastEnqueuedMessage == "bang")
```

可能会失败；你将不得不留出一些时间来传递，然后重新尝试检查`TestKit.awaitCond`。


