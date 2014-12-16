# 持久化

Akka持久化使有状态的actor能留存其内部状态，以便在因JVM崩溃、监管者引起，或在集群中迁移导致的actor启动、重启时恢复它。Akka持久化背后的关键概念是持久化的只是一个actor的内部状态的的变化，而不是直接持久化其当前状态 （除了可选的快照）。这些更改永远只能被附加到存储，没什么是可变的，这使得高事务处理率和高效复制成为可能。有状态actor通过重放保存的变化来恢复，从而使它们可以重建其内部状态。重放的可以是完整历史记录，或着从某一个快照开始从而可以大大减少恢复时间。Akka持久化也提供了“至少一次消息传递语义”的点对点通信。

> 注意

> 本模块被标记为**“experimental”**直到Akka 2.3.0引入它。我们将基于用户的反馈继续改善此API，这就意味着我们对维护版本的二进制不兼容性降到最低的保证不适用于``akka.persistence``包的内容。

Akka持久化受[eventsourced](https://github.com/eligosource/eventsourced)启发，并且是其正式的替代者。它遵循[eventsourced](https://github.com/eligosource/eventsourced)相同的概念和体系结构，但在API和实现层上则显著不同。又见[《迁移指南：从Eventsourced到Akka Persistence 2.3》](http://doc.akka.io/docs/akka/2.3.6/project/migration-guide-eventsourced-2.3.x.html#migration-eventsourced-2-3)。

###Akka 2.3.4的变化
在Akka 2.3.4中，较早版本中的几个概念被推倒和简化。大体上讲；``Processor``和``EventsourcedProcessor``被替换为``PersistentActor``。``Channel``和``PersistentChannel``被替换为``AtLeastOnceDelivery``。``View``被替换为``PersistentView``。

更改的全部细节请参阅[《迁移指南：从Akka Persistence (experimental) 2.3.3到Akka Persistence 2.3.4 (和2.4.x)》](http://doc.akka.io/docs/akka/2.3.6/project/migration-guide-persistence-experimental-2.3.x-2.4.x.html#migration-guide-persistence-experimental-2-3-x-2-4-x)。老的类在一段时间内仍被包含并标记为废弃，以便用户顺利过渡。如果你需要的旧的文档，可以参考[这里](http://doc.akka.io/docs/akka/2.3.3/scala/persistence.html)。

###依赖
Akka持久化是一个单独的jar文件。请确保你的项目中有以下依赖关系：

```
"com.typesafe.akka" %% "akka-persistence-experimental" % "2.3.6"
```

###体系结构

* *PersistentActor*：是一个持久的、有状态的actor。它能够持久化消息到一个日志，并以线程安全的方式对它们作出响应。它是可被用于执行*命令[command]*和*事件来源[event sourced]*的actor。当一个持久化的actor被启动或重新启动时，该actor会被重播日志消息，从而可以从这些消息恢复内部状态。
* *PersistentView*：一个视图是一个持久的、有状态的actor，来接收已经由另一个持久化actor写下的日志消息。视图本身并没有新的日志消息，相反，它只能从一个持久化actor复制消息流来更新内部状态。
* *AtLeastOnceDelivery*：使用至少一次的传递语义将消息发送到目的地，以防发送者和接收者 JVM崩溃。
* *Journal*：日志存储发送到一个持久化actor的消息序列。应用程序可以控制actor接收的消息中，哪些需要在日记中记录，哪些不需要。日志的存储后端是可插拔的。默认日志存储插件是写入本地文件系统，复制日志在[社区插件](http://akka.io/community/)中可以获得。
* *Snapshot store*：快照存储区持久化一个持久化actor或一个视图的内部状态的快照。快照可用于优化恢复时间。快照存储区的存储后端是可插拔的。默认快照存储插件写入本地文件系统。

###<a name="Event_sourcing"></a>事件来源 Event sourcing

[事件来源](http://martinfowler.com/eaaDev/EventSourcing.html)背后的基本思想很简单。一个持久化actor接收一个 (非持久化) 命令，它首先会被验证是否可以被应用到当前状态。在这里，验证可以意味着任何东西，例如从对命令消息字段的简单检查，到引用若干外部服务。如果验证成功，从该命令生成事件，表示命令的效果。然后这些事件被持久化，在成功的持久化后，用于改变actor的状态。当持久化actor需要恢复时，仅重播持久化的事件，因为我们知道他们可以被成功地应用。换句话说，与命令不同，被重播到一个持久化actor的事件不能失败。事件来源的actor当然也可以处理不改变应用程序状态的命令，例如查询命令。

Akka持久化通过``PersistentActor``特质支持事件来源。一个actor可以扩展这个特质来使用``persist``方法持久化和处理事件。``PersistentActor``的行为是通过实现``receiveRecover``和``receiveCommand``定义的。下面的示例演示了这一点。

```scala
import akka.actor._
import akka.persistence._

case class Cmd(data: String)
case class Evt(data: String)

case class ExampleState(events: List[String] = Nil) {
  def updated(evt: Evt): ExampleState = copy(evt.data :: events)
  def size: Int = events.length
  override def toString: String = events.reverse.toString
}

class ExamplePersistentActor extends PersistentActor {
  override def persistenceId = "sample-id-1"

  var state = ExampleState()

  def updateState(event: Evt): Unit =
    state = state.updated(event)

  def numEvents =
    state.size

  val receiveRecover: Receive = {
    case evt: Evt                                 => updateState(evt)
    case SnapshotOffer(_, snapshot: ExampleState) => state = snapshot
  }

  val receiveCommand: Receive = {
    case Cmd(data) =>
      persist(Evt(s"${data}-${numEvents}"))(updateState)
      persist(Evt(s"${data}-${numEvents + 1}")) { event =>
        updateState(event)
        context.system.eventStream.publish(event)
      }
    case "snap"  => saveSnapshot(state)
    case "print" => println(state)
  }

}
```

该示例定义了两种数据类型，``Cmd`` 和 ``Evt`` 分别代表命令和事件。``ExamplePersistentActor``的``state`` 包含在 ``ExampleState``中的持久化的事件数据的列表。

持久化actor的``receiveRecover``方法定义如何通过在恢复过程中处理 ``Evt`` 和 ``SnapshotOffer`` 消息来更新``state``。持久化actor的``receiveCommand`` 方法是一个命令处理程序。在此示例中，命令处理是通过生成两个事件，然后被持久化和处理的。事件通过调用``persist``方法持久化，该方法第一个参数是事件（或一系列事件），第二个参数是事件处理程序。

``persist``方法以异步方式持久化事件，而事件处理程序对成功持久化的事件进行处理。成功持久化的事件在内部作为独立消息发送回给持久化actor，来触发事件处理执行。事件处理程序可能会包含持久化actor的状态并修改它。持久化事件的发送者也是相应命令的发送者。这使事件处理程序可以回复命令的发送者（未显示）。

事件处理程序的主要任务是：使用事件数据更改持久化actor状态，并通过发布事件通知其他人成功的状态变化。

当使用``persist``持久化事件的时候，可以保证持久化actor在``persist``调用和相应的事件处理程序的（多次）执行之间不会进一步收到命令。这在单个命令的上下文中多次调用``persist``的情况下也成立。

运行该示例最简单的方法是下载[Typesafe Activator](http://www.typesafe.com/platform/getstarted)，并打开[Akka Persistence Samples with Scala](http://www.typesafe.com/activator/template/akka-sample-persistence-scala)这个教程。它包含如何运行``PersistentActorExample``的说明。

> 注意

> 还有可能在正常处理过程中使用不同的命令处理程序，并使用``context.become()``和``context.unbecome()``来恢复。恢复后使actor进入相同的状态，你需要特别谨慎地使用``receiveRecover``方法中的``become``和``unbecome``进行相同的状态转换，就像你会在命令处理程序中做的一样。

#####标识符
一个持久化actor必须具有跨不同actor化身而不改变的标识符。必须使用``persistenceId``方法定义该标识符。

```scala
override def persistenceId = "my-stable-persistence-id"
```

#####<a name="recovery"/>恢复
默认情况下，一个持久化actor通过在启动和重启时重放日志消息实现自动恢复。恢复过程中发送给持久化actor的新消息不会干扰重放消息。新消息只会在持久化actor恢复完成后被收到。

######自定义恢复
通过使用空实现重写``preStart``，可以禁用启动时的自动恢复。


```scala
override def preStart() = ()
```

在这种情况下，必须显式地通过``Recover()``消息的发送恢复一个持久化actor。

```scala
processor ! Recover()
```

如果没有重写，``preStart``将发送一个``Recover()``消息到``self``。应用程序还可能重写``preStart``来定义进一步的``Recover()`` 参数如序列号范围上界，例如。

```scala
override def preStart() {
  self ! Recover(toSequenceNr = 457L)
}
```

序列号范围上界可以用来恢复持久化actor到过去的某个状态，而不是当前状态。通过使用空实现重写``preRestart``，可以禁用重新启动时的自动恢复。

```scala
override def preRestart(reason: Throwable, message: Option[Any]) = ()
```

######恢复状态
一个持久化actor可以通过以下方法查询自身的恢复状态

```scala
def recoveryRunning: Boolean
def recoveryFinished: Boolean
```

有时持久化actor在恢复完成时，处理任意其他消息之前，需要执行额外的初始化。持久化actor会在恢复完成后，处理任意其他消息之前，收到一个特别的``RecoveryCompleted``消息。

如果该actor在从日志中恢复状态出现问题，该actor将发送`RecoveryFailure`消息并可以选择在`receiveRecover`中处理。如果该actor不处理`RecoveryFailure`消息，它将被停止。

```scala
def receiveRecover: Receive = {
  case RecoveryCompleted => recoveryCompleted()
  case evt               => //...
}

def receiveCommand: Receive = {
  case msg => //...
}

def recoveryCompleted(): Unit = {
  // perform init after recovery, before any other messages
  // ...
}
```

#####放宽的局部一致性要求和高吞吐量的用例

如果面临放宽的局部一致性要求和高吞吐量，有时``PersistentActor``及其``persist``在处理大量涌入的命令时可能会不够，因为它必须等待知道给定命令相关的所有事件都处理完成后，才开始处理下一条命令。虽然这种抽象在大多数的情况下非常有用，有时你可能会放宽一致性要求——例如你会想要尽可能快速地处理命令，假设事件最终会持久化并在后台恰当处理，并在需要时追溯性地回应持久性故障。

``persistAsync``方法提供了一个工具，用于实现高吞吐量的持久化actor。在日志仍在致力于持久化和（或） 执行用户事件回调代码时，它*不*会贮藏传入的命令。

在下面的示例中，事件回调可能在"任何时候"被调用，甚至在处理下一条命令之后。两个事件之间的顺序仍能得到保证（"evt-b-1"将在"evt-a-2"后发送，而它又在"evt-a-1"后发送，以此类推）。

```scala
class MyPersistentActor extends PersistentActor {

  override def persistenceId = "my-stable-persistence-id"

  def receiveRecover: Receive = {
    case _ => // handle recovery here
  }

  def receiveCommand: Receive = {
    case c: String => {
      sender() ! c
      persistAsync(s"evt-$c-1") { e => sender() ! e }
      persistAsync(s"evt-$c-2") { e => sender() ! e }
    }
  }
}

// usage
processor ! "a"
processor ! "b"

// possible order of received messages:
// a
// b
// evt-a-1
// evt-a-2
// evt-b-1
// evt-b-2
```

> 注意

> 为了实现"*命令源*"模式，只需对所有传入消息马上调用``persistAsync(cmd)(...)``，并在回调中处理它们。


> 警告

> 如果在调用``persistAsync``和日志确定写操作之间，actor被重启（或停止）时，将不会调用回调。

#####推迟行动，直到持久化处理程序已执行

使用``persistAsync``时，有时你会发现定义一些''在``persistAsync``处理程序调用之后发生''的行动是很好的。``PersistentActor``提供了一个工具方法``defer``，它类似于``persistAsync``，可是并不持久化过去的事件。推荐它用于*读取*的操作，和在你的域模型中没有相应事件的行动。

使用这种方法和持久化系列方法的使用是非常相似的，但它**不会**持久化过去的事件。它将保留在内存中，并在调用处理程序时使用。

```scala
class MyPersistentActor extends PersistentActor {

  override def persistenceId = "my-stable-persistence-id"

  def receiveRecover: Receive = {
    case _ => // handle recovery here
  }

  def receiveCommand: Receive = {
    case c: String => {
      sender() ! c
      persistAsync(s"evt-$c-1") { e => sender() ! e }
      persistAsync(s"evt-$c-2") { e => sender() ! e }
      defer(s"evt-$c-3") { e => sender() ! e }
    }
  }
}
```

注意``sender()``是可以在处理程序回调中**安全**访问的，并将指向``defer``处理程序被调用的命令的原始发送者。

调用方将以这样的顺序（保证）获得响应：

```scala
processor ! "a"
processor ! "b"

// order of received messages:
// a
// b
// evt-a-1
// evt-a-2
// evt-a-3
// evt-b-1
// evt-b-2
// evt-b-3
```

> 警告

> 如果该actor在调用``defer``和日志处理与确认所有写入之间的回调，将不会在actor重启（或停止）时调用。

#####批处理写操作
为了优化吞吐量，一个持久化actor在高负荷下，会内部将一批事件先储存，然后再（作为一个批处理）写到日志中。批处理大小可以调整，从低和中等载荷作用下的1，动态增长到高负荷下可配置的最大大小（默认为``200``）。在使用``persistAsync``时，这极大地增加了最大吞吐量。

```
akka.persistence.journal.max-message-batch-size = 200
```

只要一个batch达到最大大小或日志完成前一批写操作，就会立即触发持久化actor新的批处理写操作。批处理写操作永远不会是基于计时器的，从而将延迟保持在最低限度。

批处理也在内部使用确保事件写操作的原子性。在单个命令上下文中的所有事件将作为单个批处理写入到日志中（即使在一个命令中多次调用``persist``）。因此，``PersistentActor``的恢复将永远不会部分完成 （只持久化单个命令中事件的一个子集）。

#####删除邮件
若要删除所有消息（由一个持久化actor记录) 到指定的序列号，持久化actor可以调用``deleteMessages``方法。

一个可选的``permanent``参数指定是否应从日志中永久删除消息，或仅标记为已删除。在这两种情况下，消息都不会重播。Akka持久化以后的扩展将允许重播标记为已删除的消息，例如可用于调试。

###持久化视图

持久化视图可以通过扩展``PersistentView``特质以及实现``receive``和``persistenceId``方法实现。

```scala
class MyView extends PersistentView {
  override def persistenceId: String = "some-persistence-id"
  override def viewId: String = "some-persistence-id-view"

  def receive: Actor.Receive = {
    case payload if isPersistent =>
    // handle message from journal...
    case payload                 =>
    // handle message from user-land...
  }
}
```

``PersistenceId``标识从视图中接收的日志消息来自的持久化actor。该引用持久化actor实际并非必须正在运行。视图直接从一个持久化actor日志中读取消息。当一个持久化actor后来启动，并开始写新消息时，将默认自动更新相应的视图。

可以确定一条消息是从日志中发送，还是由用户定义的另一个调用``isPersistent``方法的actor发送。尽管有这样的功能，很多时候你根本不需要此信息，并可以简单地将相同的逻辑应用于这两种情况（跳过``if isPersistent``检查）。

#####更新
actor系统的所有视图的默认更新间隔是可配置的：

```
akka.persistence.view.auto-update-interval = 5s
```

``PersistentView``实现类还可以重写``autoUpdateInterval``方法，以返回对特定的视图类或视图实例自定义的更新时间间隔。应用程序也可以在任何时候通过对一个视图发送``Update``消息触发额外的数据更新。

```scala
val view = system.actorOf(Props[MyView])
view ! Update(await = true)
```

如果``await``参数设置为``true``，在``Update``请求后面的消息在增量消息重播时会被处理，在这个更新请求处理完成时触发。如果设置为``false``（默认值），更新请求后的消息可能与重播的消息流交织。自动更新始终以``await = false``运行。

actor系统中所有视图的自动更新可以在配置中关闭：

```
akka.persistence.view.auto-update = off
```

实现类可以通过重载``autoUpdate``方法重写配置的默认值。若要限制的每个更新请求的重播消息数量，应用程序可以配置自定义的``akka.persistence.view.auto-update-replay-max``值或重载``autoUpdateReplayMax``方法。手动更新的重播消息数目可以通过``Update``消息的``replayMax``参数进行限制。

#####恢复
持久化视图的初始化恢复过程和持久化actor的工作方式相同（即通过发送一个``Recover``消息到自己）。初始化恢复的最大重放消息数由``autoUpdateReplayMax``确定。关于自定义初始化恢复更多的可能性参见[恢复](#recovery)一节。

#####标识符
一个持久化视图必须具有跨不同actor化身而不改变的标识符。必须使用``viewId``方法定义该标识符。

``ViewId``必须不同于引用的``persistenceId``，除非[快照](#snapshots)视图和其持久化actor是共享的（即应用程序通常不需要做的东西）。

###<a name="snapshots"/>快照

快照可以大幅减少持久化actor和视图的恢复时间。下面讨论的快照内容是基于持久化actor的上下文，但这也同样适用于持久化视图。

持久化actor可以通过调用``saveSnapshot``方法保存内部状态的快照。如果快照保存成功，持久化actor接收``SaveSnapshotSuccess``消息，否则``SaveSnapshotFailure``消息

```scala
class MyProcessor extends Processor {
  var state: Any = _

  def receive = {
    case "snap"                                => saveSnapshot(state)
    case SaveSnapshotSuccess(metadata)         => // ...
    case SaveSnapshotFailure(metadata, reason) => // ...
  }
}
```

这里``metadata``的类型是``SnapshotMetadata``：

```scala
case class SnapshotMetadata(@deprecatedName('processorId) persistenceId: String, sequenceNr: Long, timestamp: Long = 0L) {
  @deprecated("Use persistenceId instead.", since = "2.3.4")
  def processorId: String = persistenceId
}
```

在恢复期间，持久化actor可以通过``SnapshotOffer``消息获取以前保存的快照，从中可以初始化内部状态。

```scala
class MyProcessor extends Processor {
  var state: Any = _

  def receive = {
    case SnapshotOffer(metadata, offeredSnapshot) => state = offeredSnapshot
    case Persistent(payload, sequenceNr)          => // ...
  }
}
```

紧随着``SnapshotOffer``的重播消息，如果有的话，是比快照年轻的。他们帮助持久化actor恢复到其当前（即最新的）状态。

一般情况下，如果持久化actor之前保存了多份快照，且这些快照中至少有一个满足``SnapshotSelectionCriteria``并可被指定用于恢复的情况下，才会给持久化actor提供一个快照。

```scala
processor ! Recover(fromSnapshot = SnapshotSelectionCriteria(
  maxSequenceNr = 457L,
  maxTimestamp = System.currentTimeMillis))
```

如果未指定，他们默认为``SnapshotSelectionCriteria.Latest``，即选择最新（= 最小）的快照。若要禁用基于快照的恢复，应用程序应使用``SnapshotSelectionCriteria.None``。如果已保存的快照没有匹配指定的``SnapshotSelectionCriteria``，恢复时将重播所有日志消息。

#####快照删除
一个持久化actor可以通过调用``deleteSnapshot``方法并指定快照的序列号与的时间戳作为参数，来删除单个快照。要批量删除匹配``SnapshotSelectionCriteria``的快照，持久化actor应该使用``deleteSnapshots``方法。

###至少一次投递
要在至少一次投递语义下发送消息到目的地，你可以在发送端的``PersistentActor``混入``AtLeastOnceDelivery``特质。如果他们在可配置的超时时间内未得到确认，它负责重新发送消息。

> 注意

> 至少一次投递意味着原始消息发送顺序并不总是保留的，以及目的地可能接收重复的消息。这意味着语义不匹配那些正常的`ActorRef`发送操作：

> * 它不是在最多一次投递
> * 同一个发件人-接收人对的消息顺序不保留，因为可重新发送
> * 崩溃并重新启动后，消息仍然会发送到目的地——向新actor化身

> 这些语义和`ActorPath`所表示的相似（见[actor生命周期](actors.md#actor-lifecycle-scala)），因此你在发送消息时需要提供的是一个路径而不是一个引用。消息被发送到一个指向actor selection的路径。

``deliver``方法用于将消息发送到目的地。当目的地已回复一条确认消息，调用``confirmDelivery``方法。

```scala
import akka.actor.{ Actor, ActorPath }
import akka.persistence.AtLeastOnceDelivery

case class Msg(deliveryId: Long, s: String)
case class Confirm(deliveryId: Long)

sealed trait Evt
case class MsgSent(s: String) extends Evt
case class MsgConfirmed(deliveryId: Long) extends Evt

class MyPersistentActor(destination: ActorPath)
  extends PersistentActor with AtLeastOnceDelivery {

  def receiveCommand: Receive = {
    case s: String           => persist(MsgSent(s))(updateState)
    case Confirm(deliveryId) => persist(MsgConfirmed(deliveryId))(updateState)
  }

  def receiveRecover: Receive = {
    case evt: Evt => updateState(evt)
  }

  def updateState(evt: Evt): Unit = evt match {
    case MsgSent(s) =>
      deliver(destination, deliveryId => Msg(deliveryId, s))

    case MsgConfirmed(deliveryId) => confirmDelivery(deliveryId)
  }
}

class MyDestination extends Actor {
  def receive = {
    case Msg(deliveryId, s) =>
      // ...
      sender() ! Confirm(deliveryId)
  }
}
```

``deliver``和``confirmDelivery``之间的相关性，是通过传入``deliveryIdToMessage``函数的``deliveryId``参数进行的。通常在消息中包含``deliveryId``传递到目的地，然后用一个包含相同``deliveryId``的消息进行答复。

``deliveryId``是无间隙严格单调递增序列号。相同的序列将用于所有目标actor，即当发送到多个目标时会看到序列中的空白，如果没有执行转译。

``AtLeastOnceDelivery``特质具有未经确认的消息和一个序列号组成的一个状态。它并不存储这个状态本身。你必须持久化从你的``PersistentActor``调用``deliver``和``confirmDelivery``所对应的事件，从而可以通过调用相同的方法在``PersistentActor``的恢复阶段恢复状态。有时这些事件可以来自其他业务级别的事件，而有时你必须创建单独的事件。在恢复过程中``deliver``的调用不会发出消息，但如果没有匹配的``confirmDelivery``执行，它将稍后发送。

支持快照功能是``getDeliverySnapshot``和``setDeliverySnapshot``提供的。``AtLeastOnceDeliverySnapshot``包含完整的投递状态，包括未经确认的消息。如果你需要一个自定义的快照保存actor其他部分的状态，你还必须包括``AtLeastOnceDeliverySnapshot``。它使用``protobuf``序列化，即利用Akka的通用序列化机制。最简单的方法是将``AtLeastOnceDeliverySnapshot``中的字节作为blob包含在你自定义的快照中。

重发尝试之间的间隔是由``redeliverInterval``方法定义的。其默认值可以用``akka.persistence.at-least-once-delivery.redeliver-interval``配置键来配置。可以在实现类中重写该方法来返回非默认值。

经过若干次尝试后，一个``AtLeastOnceDelivery.UnconfirmedWarning``消息将发送到``self``。重新发送仍会继续，但你可以选择调用``confirmDelivery``来取消重新发送。``warnAfterNumberOfUnconfirmedAttempts``方法定义发出警告之前传递尝试的次数。其默认值可以用``akka.persistence.at-least-once-delivery.warn-after-number-of-unconfirmed-attempts``配置键配置。可以用实现类重写该方法来返回非默认值。

``AtLeastOnceDelivery``特质将消息保留在内存中，直到他们成功投递已被确认。actor能保留在内存中的未经确认的消息的最大数目限制是由``maxUnconfirmedMessages``方法定义的。如果超过了此限制``deliver``方法将不会接受更多的消息，它将抛出``AtLeastOnceDelivery.MaxUnconfirmedMessagesExceededException``。可以用``akka.persistence.at-least-once-delivery.max-unconfirmed-messages``配置键配置其默认值。可以用实现类重写该方法来返回非默认值。

###存储插件
对于日志和快照存储的存储后端在Akka持久化中是可插拔的。默认日志插件将消息写入LevelDB（见[本地LevelDB日志](#local-leveldb-journal)）。默认快照存储插件将快照作为单独的文件写入本地文件系统（请参阅[本地快照存储区](#local-snapshot-store)）。应用程序可以通过实现一个插件API并通过配置激活它们来提供他们自己的插件。插件开发需要以下引入：

```scala
import akka.actor.ActorSystem
import akka.persistence._
import akka.persistence.journal._
import akka.persistence.snapshot._
import akka.testkit.TestKit
import com.typesafe.config._
import org.scalatest.WordSpec

import scala.collection.immutable.Seq
import scala.concurrent.Future
import scala.concurrent.duration._
```

#####日志插件API
日志插件要扩展``SyncWriteJournal``或``AsyncWriteJournal``。``SyncWriteJournal``是一个actor，当存储后端的API只支持同步、阻塞写入时应扩展它。在这种情况下，要实现的方法是：

```scala
/**
 * Plugin API: synchronously writes a batch of persistent messages to the journal.
 * The batch write must be atomic i.e. either all persistent messages in the batch
 * are written or none.
 */
def writeMessages(messages: immutable.Seq[PersistentRepr]): Unit

/**
 * Plugin API: synchronously writes a batch of delivery confirmations to the journal.
 */
@deprecated("writeConfirmations will be removed, since Channels will be removed.", since = "2.3.4")
def writeConfirmations(confirmations: immutable.Seq[PersistentConfirmation]): Unit

/**
 * Plugin API: synchronously deletes messages identified by `messageIds` from the
 * journal. If `permanent` is set to `false`, the persistent messages are marked as
 * deleted, otherwise they are permanently deleted.
 */
@deprecated("deleteMessages will be removed.", since = "2.3.4")
def deleteMessages(messageIds: immutable.Seq[PersistentId], permanent: Boolean): Unit

/**
 * Plugin API: synchronously deletes all persistent messages up to `toSequenceNr`
 * (inclusive). If `permanent` is set to `false`, the persistent messages are marked
 * as deleted, otherwise they are permanently deleted.
 */
def deleteMessagesTo(persistenceId: String, toSequenceNr: Long, permanent: Boolean): Unit
```

当存储后端的API支持异步、非阻塞写入时，应扩展``AsyncWriteJournal``这个actor。在这种情况下，要实现的方法是：

```scala
/**
 * Plugin API: asynchronously writes a batch of persistent messages to the journal.
 * The batch write must be atomic i.e. either all persistent messages in the batch
 * are written or none.
 */
def asyncWriteMessages(messages: immutable.Seq[PersistentRepr]): Future[Unit]

/**
 * Plugin API: asynchronously writes a batch of delivery confirmations to the journal.
 */
@deprecated("writeConfirmations will be removed, since Channels will be removed.", since = "2.3.4")
def asyncWriteConfirmations(confirmations: immutable.Seq[PersistentConfirmation]): Future[Unit]

/**
 * Plugin API: asynchronously deletes messages identified by `messageIds` from the
 * journal. If `permanent` is set to `false`, the persistent messages are marked as
 * deleted, otherwise they are permanently deleted.
 */
@deprecated("asyncDeleteMessages will be removed.", since = "2.3.4")
def asyncDeleteMessages(messageIds: immutable.Seq[PersistentId], permanent: Boolean): Future[Unit]

/**
 * Plugin API: asynchronously deletes all persistent messages up to `toSequenceNr`
 * (inclusive). If `permanent` is set to `false`, the persistent messages are marked
 * as deleted, otherwise they are permanently deleted.
 */
def asyncDeleteMessagesTo(persistenceId: String, toSequenceNr: Long, permanent: Boolean): Future[Unit]
```

消息重播和序列号恢复始终是异步的，因此任何日志插件必须实现：

```scala
/**
 * Plugin API: asynchronously replays persistent messages. Implementations replay
 * a message by calling `replayCallback`. The returned future must be completed
 * when all messages (matching the sequence number bounds) have been replayed.
 * The future must be completed with a failure if any of the persistent messages
 * could not be replayed.
 *
 * The `replayCallback` must also be called with messages that have been marked
 * as deleted. In this case a replayed message's `deleted` method must return
 * `true`.
 *
 * The channel ids of delivery confirmations that are available for a replayed
 * message must be contained in that message's `confirms` sequence.
 *
 * @param persistenceId persistent actor id.
 * @param fromSequenceNr sequence number where replay should start (inclusive).
 * @param toSequenceNr sequence number where replay should end (inclusive).
 * @param max maximum number of messages to be replayed.
 * @param replayCallback called to replay a single message. Can be called from any
 *                       thread.
 *
 * @see [[AsyncWriteJournal]]
 * @see [[SyncWriteJournal]]
 */
def asyncReplayMessages(persistenceId: String, fromSequenceNr: Long, toSequenceNr: Long, max: Long)(replayCallback: PersistentRepr ⇒ Unit): Future[Unit]

/**
 * Plugin API: asynchronously reads the highest stored sequence number for the
 * given `persistenceId`.
 *
 * @param persistenceId persistent actor id.
 * @param fromSequenceNr hint where to start searching for the highest sequence
 *                       number.
 */
def asyncReadHighestSequenceNr(persistenceId: String, fromSequenceNr: Long): Future[Long]
```

日志插件可以以下最小配置下激活：

```scala
# Path to the journal plugin to be used
akka.persistence.journal.plugin = "my-journal"

# My custom journal plugin
my-journal {
  # Class name of the plugin.
  class = "docs.persistence.MyJournal"
  # Dispatcher for the plugin actor.
  plugin-dispatcher = "akka.actor.default-dispatcher"
}
```

指定的插件``class``必须具有一个无参数构造函数。``plugin-dispatcher``是用于插件actor的调度程序。如果未指定，则默认是对``SyncWriteJournal``插件的``akka.persistence.dispatchers.default-plugin-dispatcher``和对``AsyncWriteJournal``插件的``akka.actor.default-dispatcher``。

#####快照存储插件API

一个快照存储插件必须扩展``SnapshotStore``actor并实现以下方法：

```scala
/**
 * Plugin API: asynchronously loads a snapshot.
 *
 * @param persistenceId processor id.
 * @param criteria selection criteria for loading.
 */
def loadAsync(persistenceId: String, criteria: SnapshotSelectionCriteria): Future[Option[SelectedSnapshot]]

/**
 * Plugin API: asynchronously saves a snapshot.
 *
 * @param metadata snapshot metadata.
 * @param snapshot snapshot.
 */
def saveAsync(metadata: SnapshotMetadata, snapshot: Any): Future[Unit]

/**
 * Plugin API: called after successful saving of a snapshot.
 *
 * @param metadata snapshot metadata.
 */
def saved(metadata: SnapshotMetadata)

/**
 * Plugin API: deletes the snapshot identified by `metadata`.
 *
 * @param metadata snapshot metadata.
 */

def delete(metadata: SnapshotMetadata)

/**
 * Plugin API: deletes all snapshots matching `criteria`.
 *
 * @param persistenceId processor id.
 * @param criteria selection criteria for deleting.
 */
def delete(persistenceId: String, criteria: SnapshotSelectionCriteria)
```

可通过以下最小配置激活快照存储插件：

```scala
# Path to the snapshot store plugin to be used
akka.persistence.snapshot-store.plugin = "my-snapshot-store"

# My custom snapshot store plugin
my-snapshot-store {
  # Class name of the plugin.
  class = "docs.persistence.MySnapshotStore"
  # Dispatcher for the plugin actor.
  plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"
}
```

指定的插件``class``必须具有一个无参数构造函数。``plugin-dispatcher``是用于插件actor的调度程序。如果未指定，则默认为``akka.persistence.dispatchers.default-plugin-dispatcher``。

#####插件TCK
为了帮助开发人员构建正确和高质量存储插件，我们提供技术兼容性工具包 (简称[TCK](http://en.wikipedia.org/wiki/Technology_Compatibility_Kit)）。

TCK可用于Java或Scala项目中，对Scala你需要引入``akka-persistence-tck-experimental``依赖关系：

	"com.typesafe.akka" %% "akka-persistence-tck-experimental" % "2.3.5" % "test"

要在你的测试套件中包括日志TCK的测试，只需要扩展提供的``JournalSpec``：

```scala
class MyJournalSpec extends JournalSpec {
  override val config = ConfigFactory.parseString(
    """
      |akka.persistence.journal.plugin = "my.journal.plugin"
    """.stripMargin)
}
```

我们还提供一个简单的基准测试类``JournalPerfSpec``，包括所有``JournalSpec``有的测试，还会执行日志上的一些长操作，并打印性能统计数据。虽然它并**不**旨在提供一个适当的基准测试环境，它仍可以被用于对典型的应用场景下你的日志表现进行粗略的感受。

要在你的测试套件中包括``SnapshotStore`` TCK测试，只需要扩展``SnapshotStoreSpec``：

```scala
class MySnapshotStoreSpec extends SnapshotStoreSpec {
  override val config = ConfigFactory.parseString(
    """
      |akka.persistence.snapshot-store.plugin = "my.snapshot-store.plugin"
    """.stripMargin)
}
```

在你的插件需要一些初始设置的情况下（启动模拟数据库，删除临时文件等），你可以重写``beforeAll``和``afterAll``来钩入测试生命周期：

```scala
class MyJournalSpec extends JournalSpec {
  override val config = ConfigFactory.parseString(
    """
      |akka.persistence.journal.plugin = "my.journal.plugin"
    """.stripMargin)

  val storageLocations = List(
    new File(system.settings.config.getString("akka.persistence.journal.leveldb.dir")),
    new File(config.getString("akka.persistence.snapshot-store.local.dir")))

  override def beforeAll() {
    super.beforeAll()
    storageLocations foreach FileUtils.deleteRecursively
  }

  override def afterAll() {
    storageLocations foreach FileUtils.deleteRecursively
    super.afterAll()
  }

}
```

我们*强烈建议*在你的测试套件包括这些规格，因为从头编写一个插件时，它们涵盖了广泛的，你可能会遗忘的测试用例。

###预先包装好的插件

#####<a name="local-leveldb-journal" />本地LevelDB日志
默认日志插件是``akka.persistence.journal.leveldb``，它将消息写入到本地的LevelDB实例。LevelDB文件的默认位置是当前工作目录中一个名为``journal``的目录。此位置可以由配置中指定的相对或绝对的路径更改：

```scala
akka.persistence.journal.leveldb.dir = "target/journal"
```

用这个插件，每个actor系统可运行其自己私有的LevelDB实例。

#####共享LevelDB日志
一个LevelDB实例还可以由多个actor系统（在相同或不同节点上）共享。它，例如，允许持久化actor进行故障转移到备份节点，并从备份节点继续使用共享的日志实例。

> 警告

> 共享的LevelDB实例是单点故障，因此应仅用于测试目的。高可用、带复本的日志可以从[社区插件](http://akka.io/community)中获得。

通过实例化``SharedLeveldbStore``actor可以启动一个共享的LevelDB实例。

```scala
  }
}

class MyJournal extends AsyncWriteJournal {
  def asyncWriteMessages(messages: Seq[PersistentRepr]): Future[Unit] = ???
  def asyncWriteConfirmations(confirmations: Seq[PersistentConfirmation]): Future[Unit] = ???
  def asyncDeleteMessages(messageIds: Seq[PersistentId], permanent: Boolean): Future[Unit] = ???
  def asyncDeleteMessagesTo(persistenceId: String, toSequenceNr: Long, permanent: Boolean): Future[Unit] = ???
  def asyncReplayMessages(persistenceId: String, fromSequenceNr: Long, toSequenceNr: Long, max: Long)(replayCallback: (PersistentRepr) => Unit): Future[Unit] = ???
  def asyncReadHighestSequenceNr(persistenceId: String, fromSequenceNr: Long): Future[Long] = ???
}

class MySnapshotStore extends SnapshotStore {
  def loadAsync(persistenceId: String, criteria: SnapshotSelectionCriteria): Future[Option[SelectedSnapshot]] = ???
  def saveAsync(metadata: SnapshotMetadata, snapshot: Any): Future[Unit] = ???
  def saved(metadata: SnapshotMetadata): Unit = ???
  def delete(metadata: SnapshotMetadata): Unit = ???
  def delete(persistenceId: String, criteria: SnapshotSelectionCriteria): Unit = ???
}

object PersistenceTCKDoc {
  new AnyRef {
    import akka.persistence.journal.JournalSpec

    class MyJournalSpec extends JournalSpec {
      override val config = ConfigFactory.parseString(
        """
          |akka.persistence.journal.plugin = "my.journal.plugin"
        """.stripMargin)
    }
  }
  new AnyRef {
    import akka.persistence.snapshot.SnapshotStoreSpec

    class MySnapshotStoreSpec extends SnapshotStoreSpec {
      override val config = ConfigFactory.parseString(
        """
          |akka.persistence.snapshot-store.plugin = "my.snapshot-store.plugin"
        """.stripMargin)
    }
  }
  new AnyRef {
    import java.io.File

    import akka.persistence.journal.JournalSpec
    import org.iq80.leveldb.util.FileUtils

    class MyJournalSpec extends JournalSpec {
      override val config = ConfigFactory.parseString(
        """
          |akka.persistence.journal.plugin = "my.journal.plugin"
        """.stripMargin)

      val storageLocations = List(
        new File(system.settings.config.getString("akka.persistence.journal.leveldb.dir")),
        new File(config.getString("akka.persistence.snapshot-store.local.dir")))

      override def beforeAll() {
        super.beforeAll()
        storageLocations foreach FileUtils.deleteRecursively
      }

      override def afterAll() {
        storageLocations foreach FileUtils.deleteRecursively
        super.afterAll()
      }

    }
  }
}
```

默认情况下，共享的实例将日志消息写入到当前的工作目录中一个名为``journal``的本地目录。可以通过配置更改存储位置：

```
akka.persistence.journal.leveldb-shared.store.dir = "target/shared"
```

使用共享的LevelDB存储的actor系统必须激活``akka.persistence.journal.leveldb-shared``插件。

```
akka.persistence.journal.plugin = "akka.persistence.journal.leveldb-shared"
```

这个插件必须由注入（远程的）``SharedLeveldbStore`` actor引用来进行初始化。注入是通过以actor引用作为参数调用``SharedLeveldbJournal.setStore``方法完成的。

```scala
trait SharedStoreUsage extends Actor {
  override def preStart(): Unit = {
    context.actorSelection("akka.tcp://example@127.0.0.1:2552/user/store") ! Identify(1)
  }

  def receive = {
    case ActorIdentity(1, Some(store)) =>
      SharedLeveldbJournal.setStore(store, context.system)
  }
}
```

内部日志命令（由持久化actor发送的）会缓冲直到注入完成。注入是幂等的，即只有第一次的注入被使用。

#####<a name="local-snapshot-store" />本地快照存储区

默认快照存储插件是``akka.persistence.snapshot-store.local``。它将快照文件写入本地文件系统。默认的存储位置是当前工作目录中一个名为``snapshots`` 的目录。这可以通过配置中指定的相对或绝对的路径来更改：

```
akka.persistence.snapshot-store.local.dir = "target/snapshots"
```

###自定义序列化
快照序列化和``Persistent``消息的有效载荷是可以通过Akka[序列化](serialization.md)基础架构配置的。例如，如果应用程序想要序列化

* 有效载荷的``MyPayload``类型与自定义的``MyPayloadSerializer``和
* 快照的类型``MySnapshot``与自定义的``MySnapshotSerializer``

它必须添加

```
akka.actor {
  serializers {
    my-payload = "docs.persistence.MyPayloadSerializer"
    my-snapshot = "docs.persistence.MySnapshotSerializer"
  }
  serialization-bindings {
    "docs.persistence.MyPayload" = my-payload
    "docs.persistence.MySnapshot" = my-snapshot
  }
}
```

在应用程序配置中。如果未指定，则使用默认的序列化程序。

###测试
运行测试时使用``sbt``的LevelDB默认设置，请确保在你的sbt项目中设置``fork := true``，否则你将看到一个``UnsatisfiedLinkError``。或者，你可以切换到一个LevelDB Java 端口，通过这样的设置

```
akka.persistence.journal.leveldb.native = off
```

或

```
akka.persistence.journal.leveldb-shared.store.native = off
```

在Akka配置中。LevelDB 的Java端口仅用于测试目的。

###杂项
#####状态机
状态机可以通过将``FSM``特质混入持久化actor来实现持久化。

```scala
import akka.actor.FSM
import akka.persistence.{ Persistent, Processor }

class PersistentDoor extends Processor with FSM[String, Int] {
  startWith("closed", 0)

  when("closed") {
    case Event(Persistent("open", _), counter) =>
      goto("open") using (counter + 1) replying (counter)
  }

  when("open") {
    case Event(Persistent("close", _), counter) =>
      goto("closed") using (counter + 1) replying (counter)
  }
}
```

###配置
配置中有几个属性为持久化模块使用，请参阅[参考配置](general/configuration.md#config-akka-persistence)。


