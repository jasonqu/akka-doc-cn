# 事件总线

> 注：本节未经校验，如有问题欢迎提issue

最初设想是为了提供一种向多个actor群发消息的方法，之后`EventBus`被一般化为一组实现一个简单接口的可组合的特质:

```scala
/**
 * Attempts to register the subscriber to the specified Classifier
 * @return true if successful and false if not (because it was already
 *   subscribed to that Classifier, or otherwise)
 */
def subscribe(subscriber: Subscriber, to: Classifier): Boolean

/**
 * Attempts to deregister the subscriber from the specified Classifier
 * @return true if successful and false if not (because it wasn't subscribed
 *   to that Classifier, or otherwise)
 */
def unsubscribe(subscriber: Subscriber, from: Classifier): Boolean

/**
 * Attempts to deregister the subscriber from all Classifiers it may be subscribed to
 */
def unsubscribe(subscriber: Subscriber): Unit

/**
 * Publishes the specified Event to this bus
 */
def publish(event: Event): Unit
```

> 注意

> 请注意 EventBus 不会保留发布消息的发件人。如果你需要原始发件人的引用必须在消息内部提供。

这个机制在Akka的多个地方用到，例如[事件流](#event-stream). 具体实现可以使用下面列出的特定构建工具块。

一个事件总线必须定义以下三种抽象类型：

* `Event` 所有发布到该总线上的事件的类型
* `Subscriber` 允许注册到该总线上的订阅者的类型
* `Classifier` 定义用来派发消息时选择订阅者的分类器

下面的 trait 在这些类型中仍然是泛化的，但它们必须在任何具体的实现中被定义。

###类别（Classifiers）
这里提到的类别是Akka发布包的一部分，如果没找到合适的就实现一个自己的类别，并不困难，到 [github]（http://github.com/akka/akka/tree/v2.3.6/akka-actor/src/main/scala/akka/event/EventBus.scala） 了解已有类别的实现。

#####查找分类法
最简单的分类法是给每个事件提取一个随机的类别，并为每一种类别维护一组订阅者。这可以用在收音机上选台来类比。 `LookupClassification` trait 仍然是泛化的，抽象了如何比较订阅者，以及具体分类的方法。

需要实现的方法如下:

```scala
import akka.event.EventBus
import akka.event.LookupClassification

case class MsgEnvelope(topic: String, payload: Any)

/**
 * Publishes the payload of the MsgEnvelope when the topic of the
 * MsgEnvelope equals the String specified when subscribing.
 */
class LookupBusImpl extends EventBus with LookupClassification {
  type Event = MsgEnvelope
  type Classifier = String
  type Subscriber = ActorRef

  // is used for extracting the classifier from the incoming events
  override protected def classify(event: Event): Classifier = event.topic

  // will be invoked for each event for all subscribers which registered themselves
  // for the event’s classifier
  override protected def publish(event: Event, subscriber: Subscriber): Unit = {
    subscriber ! event.payload
  }

  // must define a full order over the subscribers, expressed as expected from
  // `java.lang.Comparable.compare`
  override protected def compareSubscribers(a: Subscriber, b: Subscriber): Int =
    a.compareTo(b)

  // determines the initial size of the index data structure
  // used internally (i.e. the expected number of different classifiers)
  override protected def mapSize: Int = 128

}
```

对该实现的测试看上去像这样：

```scala
val lookupBus = new LookupBusImpl
lookupBus.subscribe(testActor, "greetings")
lookupBus.publish(MsgEnvelope("time", System.currentTimeMillis()))
lookupBus.publish(MsgEnvelope("greetings", "hello"))
expectMsg("hello")
```

这种分类法在对某个特定事件没有任何订阅者时是高效的。

#####子频道分类法
如果类别构成一个树形结构而且订阅可以不仅针对叶子结点，这种分类法可能是最合适的。 这可以类比成在安风格划分过的（多个）收音机频道中进行调台。 开发这种分类法是为了用在分类器正好是事件的JVM类，并且订阅者可能对订阅某个特定类的所有子类感兴趣的场合，但是它可以用在任何树形类别体系。

需要实现的方法如下:

```scala
import akka.util.Subclassification

class StartsWithSubclassification extends Subclassification[String] {
  override def isEqual(x: String, y: String): Boolean =
    x == y

  override def isSubclass(x: String, y: String): Boolean =
    x.startsWith(y)
}

import akka.event.SubchannelClassification

/**
 * Publishes the payload of the MsgEnvelope when the topic of the
 * MsgEnvelope starts with the String specified when subscribing.
 */
class SubchannelBusImpl extends EventBus with SubchannelClassification {
  type Event = MsgEnvelope
  type Classifier = String
  type Subscriber = ActorRef

  // Subclassification is an object providing `isEqual` and `isSubclass`
  // to be consumed by the other methods of this classifier
  override protected val subclassification: Subclassification[Classifier] =
    new StartsWithSubclassification

  // is used for extracting the classifier from the incoming events
  override protected def classify(event: Event): Classifier = event.topic

  // will be invoked for each event for all subscribers which registered
  // themselves for the event’s classifier
  override protected def publish(event: Event, subscriber: Subscriber): Unit = {
    subscriber ! event.payload
  }
}
```

对该实现的测试看上去像这样：

```scala
val subchannelBus = new SubchannelBusImpl
subchannelBus.subscribe(testActor, "abc")
subchannelBus.publish(MsgEnvelope("xyzabc", "x"))
subchannelBus.publish(MsgEnvelope("bcdef", "b"))
subchannelBus.publish(MsgEnvelope("abc", "c"))
expectMsg("c")
subchannelBus.publish(MsgEnvelope("abcdef", "d"))
expectMsg("d")
```

这种分类法在某事件没有任何订阅者时也很高效，但它使用一个保守锁来对共内部的类别缓存进行同步，所以不适合订阅关系以很高的频率变化的场合（记住通过发送第一个消息来“打开”一个类别，也将需要重新检查所有之前的订阅）。

#####扫描分类法
上一种分类法是为严格的树形多类别订阅设计的，而这个分类法是用在覆盖事件空间中可能互相重叠的非树形结构类别上的。 这可以比喻为有地理限制的（比如老旧的收音机信号传播方式）的（可能有多个）收音机频道之间进行调台。

需要实现的方法如下:

```scala
import akka.event.ScanningClassification

/**
 * Publishes String messages with length less than or equal to the length
 * specified when subscribing.
 */
class ScanningBusImpl extends EventBus with ScanningClassification {
  type Event = String
  type Classifier = Int
  type Subscriber = ActorRef

  // is needed for determining matching classifiers and storing them in an
  // ordered collection
  override protected def compareClassifiers(a: Classifier, b: Classifier): Int =
    if (a < b) -1 else if (a == b) 0 else 1

  // is needed for storing subscribers in an ordered collection
  override protected def compareSubscribers(a: Subscriber, b: Subscriber): Int =
    a.compareTo(b)

  // determines whether a given classifier shall match a given event; it is invoked
  // for each subscription for all received events, hence the name of the classifier
  override protected def matches(classifier: Classifier, event: Event): Boolean =
    event.length <= classifier

  // will be invoked for each event for all subscribers which registered themselves
  // for a classifier matching this event
  override protected def publish(event: Event, subscriber: Subscriber): Unit = {
    subscriber ! event
  }
}
```

对该实现的测试看上去像这样：

```scala
val scanningBus = new ScanningBusImpl
scanningBus.subscribe(testActor, 3)
scanningBus.publish("xyzabc")
scanningBus.publish("ab")
expectMsg("ab")
scanningBus.publish("abc")
expectMsg("abc")
```

这种分类法耗费的时间总是与订阅关系的数量成正比，而与实际匹配的数量无关。

#####Actor 分类法
这种分类法原来是专门为实现 [DeathWatch](../chapter3/01_actors.md#deathwatch-scala)开发的: 订阅者和类别都是 `ActorRef` 类型.

需要实现的方法如下:

```scala
import akka.event.ActorEventBus
import akka.event.ActorClassification
import akka.event.ActorClassifier

case class Notification(ref: ActorRef, id: Int)

class ActorBusImpl extends ActorEventBus with ActorClassifier with ActorClassification {
  type Event = Notification

  // is used for extracting the classifier from the incoming events
  override protected def classify(event: Event): ActorRef = event.ref

  // determines the initial size of the index data structure
  // used internally (i.e. the expected number of different classifiers)
  override protected def mapSize: Int = 128
}
```

对该实现的测试看上去像这样：

```scala
val observer1 = TestProbe().ref
val observer2 = TestProbe().ref
val probe1 = TestProbe()
val probe2 = TestProbe()
val subscriber1 = probe1.ref
val subscriber2 = probe2.ref
val actorBus = new ActorBusImpl
actorBus.subscribe(subscriber1, observer1)
actorBus.subscribe(subscriber2, observer1)
actorBus.subscribe(subscriber2, observer2)
actorBus.publish(Notification(observer1, 100))
probe1.expectMsg(Notification(observer1, 100))
probe2.expectMsg(Notification(observer1, 100))
actorBus.publish(Notification(observer2, 101))
probe2.expectMsg(Notification(observer2, 101))
probe1.expectNoMsg(500.millis)
```

这种分类器对事件类型仍然是泛化的，它在所有的场合下都是高效的。

###<a name="event-stream"></a>事件流
事件流是每个actor系统的主事件总线：它用来携带 [日志消息](02_logging.md#logging-scala) 和 [死信](#dead-letters)，并且也可以被用户代码使用来达到其它目的。它使用 [子频道分类法](#subchannel-classification) 使得可以向一组相关的频道进行注册 (就象 `RemoteLifeCycleMessage` 所用的那样). 以下例子演示了一个简单的订阅是如何工作的：

```scala
import akka.actor.{ Actor, DeadLetter, Props }

class Listener extends Actor {
  def receive = {
    case d: DeadLetter => println(d)
  }
}
val listener = system.actorOf(Props(classOf[Listener], this))
system.eventStream.subscribe(listener, classOf[DeadLetter])
```

#####缺省的处理器
actor系统在启动时会创建一些actor，并为其在事件流上订阅日志消息：这些是缺省的处理器，可以配置在例如``application.conf``:

```
akka {
  loggers = ["akka.event.Logging$DefaultLogger"]
}
```

这里以全路径类名列出的处理器将订阅所有配置的日志级别以上的日志事件类，并且当日志级别在运行时被修改时这些订阅关系也会同步修改:

```scala
system.eventStream.setLogLevel(Logging.DebugLevel)
```

这意味着一个低于日志级别的日志事件，事实上根本不会被派发（除非专门为相应的事件类进行了手工订阅）

#####<a name="dead-letters"></a>死信
正如 [终止actors](../chapter3/01_actors.md#stopping-actors-scala)所述, 当actor终止后其邮箱队列中的剩余消息及后续被发送的消息都将被发送到死信邮箱， 它缺省情况下会发布打包在 `DeadLetter` 中的消息. 这种打包动作会保留被重定向消息的原始发送者、接收者以及消息内容。

#####其它用处
事件流一直存在并可用，你可以向它发布你自己的事件 (它接受 ``AnyRef``) 并对相应的JVM类添加订阅监听器。

