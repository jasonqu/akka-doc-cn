# 持久化

Akka持久化使有状态的actor能留存其内部状态，以便在因JVM崩溃、监管者引起，或在集群中迁移导致的actor启动、重启时恢复它。Akka持久化背后的关键概念是持久化的只是一个actor的内部状态的的变化，而不是直接持久化其当前状态 （除了可选快照）。这些更改永远只能被附加到存储，没什么是可变的，这使得高事务处理率和高效复制成为可能。有状态actor通过重放保存的变化来恢复，从而使它们可以重建其内部状态。重放的可以是完整历史记录，或着从某一个快照开始从而可以大大减少恢复时间。Akka持久化也提供了“至少一次消息传递语义”的点对点通信。

> 注意

> 本模块被标记为**“experimental”**直到Akka 2.3.0引入它。我们将基于用户的反馈继续改善此API，这就意味着我们对维护版本的二进制不兼容性降到最低的保证不适用于``akka.persistence``包的内容。

Akka持久化受[eventsourced](https://github.com/eligosource/eventsourced)启发，并且是其正式的替代者。它遵循[eventsourced](https://github.com/eligosource/eventsourced)相同的概念和体系结构，但在API和实现层上则显著不同。又见[《迁移指南：从Eventsourced到Akka Persistence 2.3》](http://doc.akka.io/docs/akka/2.3.6/project/migration-guide-eventsourced-2.3.x.html#migration-eventsourced-2-3)。 TODO

###Akka 2.3.4的变化
在Akka 2.3.4中，较早版本中的几个概念被推倒和简化。大体上讲 ；``Processor``和``EventsourcedProcessor``被替换为``PersistentActor``。``Channel``和``PersistentChannel``被替换为``AtLeastOnceDelivery``。``View``被替换为``PersistentView``。

更改的全部细节请参阅[《迁移指南：从Akka Persistence (experimental) 2.3.3到Akka Persistence 2.3.4 (和2.4.x)》](http://doc.akka.io/docs/akka/2.3.6/project/migration-guide-persistence-experimental-2.3.x-2.4.x.html#migration-guide-persistence-experimental-2-3-x-2-4-x)。老的类在一段时间内仍被包含并标记为废弃，以便用户顺利过渡。如果你需要的旧的文档，可以参考[这里](http://doc.akka.io/docs/akka/2.3.3/scala/persistence.html)。

###依赖
Akka持久化是一个单独的jar文件。请确保你的项目中有以下依赖关系：

```
"com.typesafe.akka" %% "akka-persistence-experimental" % "2.3.6"
```

###体系结构

* *PersistentActor*：是一个持久的、有状态的actor。它能够持久化消息到一个日志，并以线程安全的方式对它们作出响应。它可被用于执行*命令[command]*和*事件来源[event sourced]*的actor。当一个持久化的actor被启动或重新启动时，该actor会被重播日志消息，从而可以从这些消息恢复内部状态。
* *PersistentView*：一个视图是一个持久的、有状态的actor，来接收已经由另一个持久化actor写下的日志消息。视图本身并没有新的日志消息，相反，它只能从一个持久化actor复制消息流来更新内部状态。
* *AtLeastOnceDelivery*：使用至少一次的传递语义将消息发送到目的地，以防发送者和接收者 JVM崩溃。
* *Journal*：日志存储发送到一个持久化actor的消息序列。应用程序可以控制actor接收的消息中，哪些需要在日记中记录，哪些不需要。日志的存储后端是可插拔的。默认日志存储插件是写入本地文件系统，复制日志在[社区插件](http://akka.io/community/)中可以获得。
* *Snapshot store*：快照存储区持久化一个持久化actor或一个视图的内部状态的快照。快照可用于优化恢复时间。快照存储区的存储后端是可插拔的。默认快照存储插件写入本地文件系统。

###<a name="Event_sourcing"></a>事件来源 Event sourcing

[事件来源](http://martinfowler.com/eaaDev/EventSourcing.html)背后的基本思想很简单。一个持久化actor接收一个 (非持久化) 命令，它首先会被验证是否可以被应用到当前状态。在这里，验证可以意味着任何东西，例如从对命令消息字段的简单检查，到引用若干外部服务。如果验证成功，从该命令生成事件，表示命令的效果。然后这些事件被持久化，在成功的持久化后，用于改变actor的状态。当持久化actor需要恢复时，仅重播持久化的事件，因为我们知道他们可以被成功地应用。换句话说，与命令不同，被重播到一个持久化actor的事件不能失败。事件来源的actor当然也可以处理不改变应用程序状态的命令，例如查询命令。

Akka持久化通过``PersistentActor``特质支持事件来源。一个actor可以扩展这个特质来使用``persist``方法持久化和处理事件。``PersistentActor``的行为是通过实现``receiveRecover``和``receiveCommand``定义的。下面的示例演示了这一点。










