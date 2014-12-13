# 路由
消息可以通过路由器发送，以便有效地将它们路由到目的actor，称为其*routee*。一个``Router``可以在actor内部或外部使用，并且你可以自己管理routee或使用有配置功能的自我包含的路由actor。

根据你应用程序的需求，可以使用不同的路由策略。Akka附带了几个有用的路由策略，开箱即用。但是正如将在这一章中看到地，你也可以[创建自己的路由](#custom-router-scala)。

<span id="simple-router-scala"></span>
###一个简单的路由器
下面的示例阐释如何使用``Router``和在actor内管理routee。

```scala
import akka.routing.ActorRefRoutee
import akka.routing.Router
import akka.routing.RoundRobinRoutingLogic
 
class Master extends Actor {
  var router = {
    val routees = Vector.fill(5) {
      val r = context.actorOf(Props[Worker])
      context watch r
      ActorRefRoutee(r)
    }
    Router(RoundRobinRoutingLogic(), routees)
  }
 
  def receive = {
    case w: Work =>
      router.route(w, sender())
    case Terminated(a) =>
      router = router.removeRoutee(a)
      val r = context.actorOf(Props[Worker])
      context watch r
      router = router.addRoutee(r)
  }
}
```

我们创建一个``Router``，并指定当路由消息到routee时，它应该使用``RoundRobinRoutingLogic``。

Akka自带的路由逻辑如下：

* ``akka.routing.RoundRobinRoutingLogic``
* ``akka.routing.RandomRoutingLogic``
* ``akka.routing.SmallestMailboxRoutingLogic``
* ``akka.routing.BroadcastRoutingLogic``
* ``akka.routing.ScatterGatherFirstCompletedRoutingLogic``
* ``akka.routing.TailChoppingRoutingLogic``
* ``akka.routing.ConsistentHashingRoutingLogic``

我们像在``ActorRefRoutee``包装下创建普通子actor一样创建routee。我们监控routee从而能够在它们被终止的情况下取代他们。

通过路由器发送消息是用``route``方法完成的，像上面例子中的``Work``消息一样。

``Router``是不可变的，而``RoutingLogic``是线程安全的；意味着他们也可以在actor外部使用。

> 注意

> 一般情况下，任何发送到路由器的消息将被向前发送到它的routee，但有一个例外。特别地[广播消息](#broadcast-messages-scala)将发送到路由器下*所有*的routee

###一个路由actor
一个路由器也可以被创建为一个自包含的actor，来管理routee，载入路由逻辑和其他配置设置。

这种类型的路由actor有两种不同的模式：

* 池——路由器创建routee作为子actor，并在该子actor终止时将它从路由器中移除。
* 群组——routee actor在路由器外部创建，而路由器将通过使用actor选择将消息发送到指定路径，而不监控其终止。

路由actor可以通过在配置中或以编程方式被定义。虽然路由actor可以在配置文件中定义，但仍然必须以编程方式创建，即你不能只通过外部配置创建路由器。如果你在配置文件中定义路由actor，则实际将使用这些设置，而不是以编程方式提供的参数。

你可以通过路由actor向routee 发送消息，就像向普通actor发消息一样，即通过其``ActorRef``。路由actor转发消息给routee时不会更改原始发件人。当routee答复路由消息时，回复将发送到原始发件人，而不是路由actor。

> 注意

> 一般地，任何发送到路由器的消息将被向前发送到它的routee，但也有几个例外。这些记录在下面[特殊处理消息](#router-special-messages-scala)一节中。

#####池
下面的代码和配置片段展示了如何创建一个将消息转发给五个``Worker`` routee的[轮循(`round-robin`)路由器](#round-robin-router-scala)。Routees 将被创建为路由器的子actor。

```
akka.actor.deployment {
  /parent/router1 {
    router = round-robin-pool
    nr-of-instances = 5
  }
}
```

```scala
val router1: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router1")
```

这里是相同的例子，但路由配置是以编程方式而不是从配置获取。

```scala
val router2: ActorRef =
  context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router2")
```

#####远程部署的Routee
除了能够创建本地actor作为routee，你可以指示路由器来部署其子actor到一系列远程主机上。Routee将以轮循方式被部署。若要远程部署routee，将路由配置包在一个``RemoteRouterConfig``下，附加要部署到的节点的远程地址。远程部署要求``akka-remote``模块被包含在类路径中。

```scala
import akka.actor.{ Address, AddressFromURIString }
import akka.remote.routing.RemoteRouterConfig
val addresses = Seq(
  Address("akka.tcp", "remotesys", "otherhost", 1234),
  AddressFromURIString("akka.tcp://othersys@anotherhost:1234"))
val routerRemote = system.actorOf(
  RemoteRouterConfig(RoundRobinPool(5), addresses).props(Props[Echo]))
```

#####发送者
默认情况下，当一个routee发送一条消息，它将[隐式地设置自身为发送者](01_actors.md#actors-tell-sender-scala)。

```scala
sender() ! x // replies will go to this actor
```

然而，通常为routee将*路由器*设置为发送者更有用。例如，你可能想要将路由器设置为发件人，如果你想要隐藏在路由器后面routee的细节。下面的代码片段演示如何设置父路由器作为发送者。

```scala
sender().tell("reply", context.parent) // replies will go back to parent
sender().!("reply")(context.parent) // alternative syntax (beware of the parens!)
```

#####监管
由池路由器创建的routee将成为路由器的孩子。因此，路由器也是子actor的监管者。

可以用该池的``supervisorStrategy``属性配置路由器actor的监管策略。如果没有提供配置，路由器的默认策略是"总是上溯"。这意味着错误都会向上传递给路由器的监管者进行处理。路由器的监管者将决定对任何错误该做什么。

请注意路由器的监管者将把错误视为路由器本身的错误。因此一个指令，用于停止或重新启动将导致路由器*本身*以停止或重新启动。此路由器，相应地，将导致它的孩子停止并重新启动。

应该提到的是路由器重新启动行为已被重写，以便重新启动时，仍重新创建这些孩子，并会在池中保留相同数量的actor。

这意味着如果你还没有指定路由器或其父节点中的`supervisorStrategy`，routee的失败会上升到路由器，并将在默认情况下重新启动路由器，从而将重新启动所有的routee（它使用上升，并在重新启动过程中不停止routee）。原因是为了使类似在子actor定义中添加`.withRouter`这样的默认行为，不会更改应用于子actor的监管策略。你可以在定义路由器时指定策略来避免低效。

设置策略是很容易完成的：

```scala
val escalator = OneForOneStrategy() {
  case e ⇒ testActor ! e; SupervisorStrategy.Escalate
}
val router = system.actorOf(RoundRobinPool(1, supervisorStrategy = escalator).props(
  routeeProps = Props[TestActor]))
```

<a name="note-router-terminated-children-scala"></a>
> 注意

> 如果路由器池的子actor终止，池路由器不会自动产生一个新的actor。在池路由器所有子actor都终止的事件中，路由器将终止本身，除非它是一个动态的路由器，例如使用了大小调整。

#####群组
有时候，相比于由路由actor创建其routee，我们更希望单独创建 routee，并提供路由器供其使用。你可以通过将 routee路径传递给路由器的配置来实现。消息将通过``ActorSelection``发送到这些路径。

下面的示例演示如何通过提供三个routee actor的路径字符串来创建一个路由器。

```
akka.actor.deployment {
  /parent/router3 {
    router = round-robin-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
val router3: ActorRef =
  context.actorOf(FromConfig.props(), "router3")
```

这里是相同的例子，但路由配置是以编程方式而不是从配置获取。

```scala
val router4: ActorRef =
  context.actorOf(RoundRobinGroup(paths).props(), "router4")
```

Routee actor将在路由器外部被创建：

```scala
system.actorOf(Props[Workers], "workers")
```

```scala
class Workers extends Actor {
  context.actorOf(Props[Worker], name = "w1")
  context.actorOf(Props[Worker], name = "w2")
  context.actorOf(Props[Worker], name = "w3")
  // ...
```

在路径中可能包含为actor在远程主机上运行的协议和地址信息。远程部署要求``akka-remote``模块被包含在类路径中。

###路由器使用
在本节中，我们将描述如何创建不同类型的路由actor。

本节中的路由actor将由名为``parent``的顶级actor创建。请注意在配置中的部署路径以``/parent/``开头，并紧接着路由actor的名字。

```scala
system.actorOf(Props[Parent], "parent")
```

<span id="round-robin-router-scala"></span>
#####RoundRobinPool 和 RoundRobinGroup
对其routee使用[轮循机制(round-robin)](http://en.wikipedia.org/wiki/Round-robin)轮询。

在配置中定义的RoundRobinPool：

```
akka.actor.deployment {
  /parent/router1 {
    router = round-robin-pool
    nr-of-instances = 5
  }
}
```

```scala
val router1: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router1")
```

在代码中定义的RoundRobinPool：

```scala
val router2: ActorRef =
  context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router2")
```

在配置中定义的RoundRobinGroup：

```
akka.actor.deployment {
  /parent/router3 {
    router = round-robin-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
val router3: ActorRef =
  context.actorOf(FromConfig.props(), "router3")
```

在代码中定义的RoundRobinGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router4: ActorRef =
  context.actorOf(RoundRobinGroup(paths).props(), "router4")
```

#####RandomPool 和 RandomGroup
该路由器类型会对每一条消息随机选择其routee。

在配置中定义的RandomPool：

```
akka.actor.deployment {
  /parent/router5 {
    router = random-pool
    nr-of-instances = 5
  }
}
```

```scala
val router5: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router5")
```

在代码中定义的RandomPool：

```scala
val router6: ActorRef =
  context.actorOf(RandomPool(5).props(Props[Worker]), "router6")
```

在配置中定义的RandomGroup：

```
akka.actor.deployment {
  /parent/router7 {
    router = random-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
val router7: ActorRef =
  context.actorOf(FromConfig.props(), "router7")
```

在代码中定义的RandomGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router8: ActorRef =
  context.actorOf(RandomGroup(paths).props(), "router8")
```

<span id="balancing-pool-scala"></span>
#####BalancingPool
将尝试重新从繁忙routee分配任务到空闲routee的路由器。所有routee都共享同一个邮箱。

在配置中定义的BalancingPool：

```
akka.actor.deployment {
  /parent/router9 {
    router = balancing-pool
    nr-of-instances = 5
  }
}
```

```scala
val router9: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router9")
```

在代码中定义的BalancingPool：

```scala
val router10: ActorRef =
  context.actorOf(BalancingPool(5).props(Props[Worker]), "router10")
```

平衡调度器的额外配置，被池使用，可以通过路由部署配置的``pool-dispatcher``节进行设定。

```scala
akka.actor.deployment {
  /parent/router9b {
    router = balancing-pool
    nr-of-instances = 5
    pool-dispatcher {
      attempt-teamwork = off
    }
  }
}
```

对BalancingPool没有群组变体。

#####SmallestMailboxPool
试图向邮箱中有最少消息的非暂停子routee发送消息的路由器。按此顺序进行选择：

* 挑选有空邮箱的空闲routee（即没有处理消息）
* 选择任一空邮箱routee
* 选择邮箱中有最少挂起消息的routee
* 选择任一远程routee，远程actor考虑优先级最低，因为其邮箱大小未知

在配置中定义的SmallestMailboxPool：

```
akka.actor.deployment {
  /parent/router11 {
    router = smallest-mailbox-pool
    nr-of-instances = 5
  }
}
```

```scala
val router11: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router11")
```

在代码中定义的SmallestMailboxPool：

```scala
val router12: ActorRef =
  context.actorOf(SmallestMailboxPool(5).props(Props[Worker]), "router12")
```

SmallestMailboxPool没有群组变体，因为邮箱大小和actor的内部调度状态从routee的路径看实际上是不可用的。

#####BroadcastPool 和 BroadcastGroup
广播的路由器将接收到的消息转发到它*所有*的routee。

在配置中定义的BroadcastPool：

```
akka.actor.deployment {
  /parent/router13 {
    router = broadcast-pool
    nr-of-instances = 5
  }
}
```

```scala
val router13: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router13")
```

在代码中定义的BroadcastPool：

```scala
val router14: ActorRef =
  context.actorOf(BroadcastPool(5).props(Props[Worker]), "router14")
```

在配置中定义的BroadcastGroup：

```
akka.actor.deployment {
  /parent/router15 {
    router = broadcast-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
val router15: ActorRef =
  context.actorOf(FromConfig.props(), "router15")
```

在代码中定义的BroadcastGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router16: ActorRef =
  context.actorOf(BroadcastGroup(paths).props(), "router16")
```

> 注意

> Broadcast路由器总是向其routee广播*每一条*消息。如果你不想播出每条消息，则你可以使用非广播路由器并使用所需的[广播消息](#broadcast-messages-scala)。

#####ScatterGatherFirstCompletedPool 和 ScatterGatherFirstCompletedGroup
ScatterGatherFirstCompletedRouter将会把消息发送到它所有的routee。然后它等待直到收到第一个答复。该结果将发送回原始发送者。其他的答复将被丢弃。

在配置的时间内，它期待至少一个答复，否则它将回复一个包含``akka.pattern.AskTimeoutException``的``akka.actor.Status.Failure``。

在配置中定义的ScatterGatherFirstCompletedPool：

```
akka.actor.deployment {
  /parent/router17 {
    router = scatter-gather-pool
    nr-of-instances = 5
    within = 10 seconds
  }
}
```

```scala
val router17: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router17")
```

在代码中定义的ScatterGatherFirstCompletedPool：

```scala
val router18: ActorRef =
  context.actorOf(ScatterGatherFirstCompletedPool(5, within = 10.seconds).
    props(Props[Worker]), "router18")
```

在配置中定义的ScatterGatherFirstCompletedGroup：

```
akka.actor.deployment {
  /parent/router19 {
    router = scatter-gather-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    within = 10 seconds
  }
}
```

```scala
val router19: ActorRef =
  context.actorOf(FromConfig.props(), "router19")
```

在代码中定义的ScatterGatherFirstCompletedGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router20: ActorRef =
  context.actorOf(ScatterGatherFirstCompletedGroup(paths,
    within = 10.seconds).props(), "router20")
```

#####TailChoppingPool 和 TailChoppingGroup
TailChoppingRouter 将首先发送消息到一个随机挑取的routee，短暂的延迟后发给第二个routee（从剩余的routee中随机挑选），以此类推。它等待第一个答复，并将它转回给原始发送者。其他答复将被丢弃。

此路由器的目标是通过查询到多个routee来减少延迟，假设其他的actor之一仍可能比第一个actor更快响应。

Peter Bailis很好地在一篇博客文章中描述了这个优化： [做冗余的工作，以加快分布式查询](http://www.bailis.org/blog/doing-redundant-work-to-speed-up-distributed-queries/)。

在配置中定义的TailChoppingPool：

```
akka.actor.deployment {
  /parent/router21 {
    router = tail-chopping-pool
    nr-of-instances = 5
    within = 10 seconds
    tail-chopping-router.interval = 20 milliseconds
  }
}
```

```scala
val router21: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router21")
```

在代码中定义的TailChoppingPool：

```scala
val router22: ActorRef =
  context.actorOf(TailChoppingPool(5, within = 10.seconds, interval = 20.millis).
    props(Props[Worker]), "router22")
```

在配置中定义的TailChoppingGroup：

```
akka.actor.deployment {
  /parent/router23 {
    router = tail-chopping-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    within = 10 seconds
    tail-chopping-router.interval = 20 milliseconds
  }
}
```

```scala
val router23: ActorRef =
  context.actorOf(FromConfig.props(), "router23")
```

在代码中定义的TailChoppingGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router24: ActorRef =
  context.actorOf(TailChoppingGroup(paths,
    within = 10.seconds, interval = 20.millis).props(), "router24")
```

#####ConsistentHashingPool 和 ConsistentHashingGroup
ConsistentHashingPool基于已发送的消息使用[一致性哈希(consistent hashing)](http://en.wikipedia.org/wiki/Consistent_hashing)选择routee。[这篇文章](http://weblogs.java.net/blog/tomwhite/archive/2007/11/consistent_hash.html)给出了如何实现一致性哈希非常好的见解。

有三种方式来定义使用哪些数据作为一致的散列键。

* 你可以定义路由的``hashMapping``，将传入的消息映射到它们一致散列键。这使决策对发送者透明。
* 这些消息可能会实现``akka.routing.ConsistentHashingRouter.ConsistentHashable``。键是消息的一部分，并很方便地与消息定义一起定义。
* 消息可以被包装在一个``akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope``中，来定义哪些数据可以用来做一致性哈希。发送者知道要使用的键。

这些定义一致性哈希键的方法，可以同时对一个路由器在一起使用。``hashMapping``被第一个尝试。

代码示例：

```scala
import akka.actor.Actor
import akka.routing.ConsistentHashingRouter.ConsistentHashable
 
class Cache extends Actor {
  var cache = Map.empty[String, String]
 
  def receive = {
    case Entry(key, value) => cache += (key -> value)
    case Get(key)          => sender() ! cache.get(key)
    case Evict(key)        => cache -= key
  }
}
 
case class Evict(key: String)
 
case class Get(key: String) extends ConsistentHashable {
  override def consistentHashKey: Any = key
}
 
case class Entry(key: String, value: String)
```

```scala
import akka.actor.Props
import akka.routing.ConsistentHashingPool
import akka.routing.ConsistentHashingRouter.ConsistentHashMapping
import akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope
 
def hashMapping: ConsistentHashMapping = {
  case Evict(key) => key
}
 
val cache: ActorRef =
  context.actorOf(ConsistentHashingPool(10, hashMapping = hashMapping).
    props(Props[Cache]), name = "cache")
 
cache ! ConsistentHashableEnvelope(
  message = Entry("hello", "HELLO"), hashKey = "hello")
cache ! ConsistentHashableEnvelope(
  message = Entry("hi", "HI"), hashKey = "hi")
 
cache ! Get("hello")
expectMsg(Some("HELLO"))
 
cache ! Get("hi")
expectMsg(Some("HI"))
 
cache ! Evict("hi")
cache ! Get("hi")
expectMsg(None)
```

在上面的例子中可以看到``Get``消息自己实现了``ConsistentHashable``，而``Entry``消息包裹在一个``ConsistentHashableEnvelope``中。``Evict``消息由``hashMapping``偏函数处理。

在配置中定义的ConsistentHashingPool：

```
akka.actor.deployment {
  /parent/router25 {
    router = consistent-hashing-pool
    nr-of-instances = 5
    virtual-nodes-factor = 10
  }
}
```

```scala
val router25: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router25")
```

在代码中定义的ConsistentHashingPool：

```scala
val router26: ActorRef =
  context.actorOf(ConsistentHashingPool(5).props(Props[Worker]),
    "router26")
```

在配置中定义的ConsistentHashingGroup：

```
akka.actor.deployment {
  /parent/router27 {
    router = consistent-hashing-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    virtual-nodes-factor = 10
  }
}
```

```scala
val router27: ActorRef =
  context.actorOf(FromConfig.props(), "router27")
```

在代码中定义的ConsistentHashingGroup：

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router28: ActorRef =
  context.actorOf(ConsistentHashingGroup(paths).props(), "router28")
```

``virtual-nodes-factor``(虚拟节点因子)是每个routee的虚拟节点数，用来在一致性哈希节点环中使用，使分布更加均匀。

<span id="router-special-messages-scala"></span>
###特殊处理的消息
发送到路由actor的大多数消息将根据路由器的路由逻辑进行转发。然而，有几种具有特殊行为的消息类型。

请注意这些特别的消息，除了``Broadcast``消息之外，只被自我包含的路由actor处理，而不是[简单的路由器](#simple-router-scala)中描述的``akka.routing.Router``组件。

<span id="broadcast-messages-scala"></span>
#####广播消息
``Broadcast``消息可用于向路由器的*所有*routee发送一条消息。当路由器接收一个``Broadcast``消息时，它将把消息的*有效载荷*发给所有的routee，不管该路由器通常如何路由其消息。

下面的示例演示了如何使用``Broadcast``消息向路由器下的每个routee发送一个非常重要的信息。

```scala
import akka.routing.Broadcast
router ! Broadcast("Watch out for Davy Jones' locker")
```

在此示例中路由器接收的广播消息，提取其有效载荷（``"Watch out for Davy Jones' locker"``），然后发送到有效载荷路由器的所有routee。它是由每个 routee actor来处理接收到的有效负载消息。

#####PoisonPill消息
PoisonPill消息对所有actor，包括路由actor，都有特殊的处理。当任何一个actor收到``PoisonPill``消息时，该actor将被停止。有关详细信息请参见[PoisonPill](01_actors.md#poison-pill-scala)。

```scala
import akka.actor.PoisonPill
router ! PoisonPill
```

路由器通常将消息传递给routee，但对于``PoisonPill``消息，需要认识到的很重要一点是它只被路由器处理。发送到路由器的``PoisonPill``消息将*不会*发送到routee。

然而，发送到路由器的``PoisonPill``消息可能仍会影响其routee，因为路由器停止时它也会停止其子actor。停止子actor是普通的actor行为。路由器将会停止其作为子actor创建的各个routee。每个孩子将处理其当前的消息，然后停止。这可能会导致一些消息未被处理。停止actor的详细信息，请参阅[文档](01_actors.md#stopping-actors-scala)。

如果你希望停止路由器及其routee，但你希望routee在停止前先处理目前在其邮箱中的所有消息，则你不应该发送``PoisonPill``消息。相反，你应该将``PoisonPill``包装在一个``Broadcast``，以便每个routee都能收到``PoisonPill``消息。请注意这将停止所有的routee，即使routee不是路由器的孩子，也就是即使是通过编程方式提供给router的routee。

```scala
import akka.actor.PoisonPill
import akka.routing.Broadcast
router ! Broadcast(PoisonPill)
```

如上代码所示，每个routee将收到一个``PoisonPill``消息。每个routee会继续如常处理其邮件，最终处理``PoisonPill``。这将导致routee停止。所有routee停止后，路由器将[自动停止](#note-router-terminated-children-scala)自己，除非它是一个动态的路由器，例如尺寸调整器。

> 注意

> Brendan W McAdams的优秀博客文章[“分布化Akka工作负载——以及完成后的关闭”](http://blog.evilmonkeylabs.com/2013/01/17/Distributing_Akka_Workloads_And_Shutting_Down_After/)更详细地讨论了如何使用``PoisonPill``消息来关闭路由器和routee。

#####Kill消息
``Kill``消息是另一种需要特殊处理的消息类型。请参阅[“如何杀掉一个actor”](01_actors.md#killing-actors-scala)来获取actor如何处理``Kill``消息的一般信息。

当``Kill``消息被发送到路由器，路由器将内部处理该消息，并且*不会*将它发送到其routee。路由器将抛出``ActorKilledException``并失败。然后它将被恢复、 重新启动或终止，取决于它如何被监督。

路由器的子routee亦将暂停，并将受应用在路由器上的监管指令影响。对不是路由器的孩子的Routee，即那些在路由器外部被创建的，将不受影响。

```scala
import akka.actor.Kill
router ! Kill
```

相比于``PoisonPill``消息，杀死一个路由器，间接杀死其子（即那些routee），和直接杀死routee（其中有些未必是其孩子）之间是有明显区别的。要直接杀死routee，路由器应发送包裹着``Kill``消息的``Broadcast``消息。

```scala
import akka.actor.Kill
import akka.routing.Broadcast
router ! Broadcast(Kill)
```

#####Managagement消息
* 发送``akka.routing.GetRoutees``到一个路由actor，使其回送一个包含当前使用routee的``akka.routing.Routees``消息。
* 发送``akka.routing.AddRoutee``到一个路由actor会将那个routee添加到其routee集合中。
* 发送``akka.routing.RemoveRoutee``到一个路由actor将从其routee集合删除该routee。
* 发送``akka.routing.AdjustPoolSize``到一个池路由actor将从其routee集合中添加或删除该数目的routee。

这些管理消息可能晚于其他消息处理，所以如果你发送``AddRoutee``后立即发送普通消息，并不能保证当普通消息被路由时，routee已被更改。如果你需要知道更改何时生效，你可以发送``AddRoutee``紧跟着``GetRoutees``，当你收到``Routees``答复，你就知道前面的变化已被应用。

<span id="resizable-routers-scala"></span>
###动态改变大小的池
大多数池可以使用固定数量的routee或有一个调整策略来动态调整routee数。

在配置中定义的包含resizer的池：

```
akka.actor.deployment {
  /parent/router29 {
    router = round-robin-pool
    resizer {
      lower-bound = 2
      upper-bound = 15
      messages-per-resize = 100
    }
  }
}
```

```scala
val router29: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router29")
```

更多选项在[配置](../chapter2/09_configuration.md)的``akka.actor.deployment.default.resizer``部分中有描述。

在代码中定义的包含resizer的池：

```scala
val resizer = DefaultResizer(lowerBound = 2, upperBound = 15)
val router30: ActorRef =
  context.actorOf(RoundRobinPool(5, Some(resizer)).props(Props[Worker]),
    "router30")
```

*需要指出如果你在配置文件中定义了``router``，那么这个值将比在代码中传入的参数有更高的优先级。*

> 注意

> 改变大小的行为是通过向actor池发送消息来触发的，但它不是完全同步的；而是向``RouterActor``的“head”发送消息来执行修改。所以在别的actor忙碌时，你不能假设改变大小的操作会立即创建新的工作actor，因为消息会被发到忙碌actor的邮箱中排队。要解决这个问题，配置actor池使用一个平衡的派发器，更多信息见[Configuring Dispatchers](#configuring-dispatchers)。

<span id="router-design-scala"></span>
###Akka中的路由是如何设计的
从表面看，路由器就像普通的actor，但是它们实际实现是不同的。路由器在收消息和快速发消息给routee被设计的极度优化。

一个普通的actor可以用来路由消息，但是actor的单线程处理会成为一个瓶颈。路由器可以通过优化原有消息处理pipeline来支持多线程，从而达到更高的吞吐量。这里是通过直接嵌入路由逻辑到其``ActorRef``，而不是在路由actor本身。发送到路由器``ActorRef``的消息可以直接被路由到routee，从而完全跳过单线程的路由actor。

当然，这个改进的成本是路由代码的内部构造相比于使用普通actor构造来说复杂许多。幸运的是所有这种复杂性对于路由API消费者来说是不可见的。然而，这却是你在实现自己的路由器时需要意识到的。

<span id="custom-router-scala"></span>
###自定义路由actor
如果觉得Akka自带的路由actor都不合用，你也可以创建自己的路由actor。要创建自己的路由，你需要满足本节中所列出的条件。

在创建你自己的路由器之前，你应该考虑一个拥有类似路由器行为的普通actor是否能完成一个成熟路由器的功能。正如[上文](#router-design-scala)解释，路由器相比于普通actor主要好处是他们拥有更高的性能。但相比普通actor他们的代码也更为复杂。因此，如果在你的应用程序中较低的最大吞吐量是可以接受的，则不妨继续使用传统的actor。不过这一节假定你想要获得最大性能，并因而演示如何创建你自己的路由器。

在此示例中创建的路由器将把每个消息复制到几个目的地。

首先从路由逻辑开始：

```scala
import scala.collection.immutable
import scala.concurrent.forkjoin.ThreadLocalRandom
import akka.routing.RoundRobinRoutingLogic
import akka.routing.RoutingLogic
import akka.routing.Routee
import akka.routing.SeveralRoutees
 
class RedundancyRoutingLogic(nbrCopies: Int) extends RoutingLogic {
  val roundRobin = RoundRobinRoutingLogic()
  def select(message: Any, routees: immutable.IndexedSeq[Routee]): Routee = {
    val targets = (1 to nbrCopies).map(_ => roundRobin.select(message, routees))
    SeveralRoutees(targets)
  }
}
```

在这个例子中``select``将被每个消息调用来使用轮询来挑选几个目的地，通过重用现有的``RoundRobinRoutingLogic``并将结果包装在一个``SeveralRoutees``实例中。``SeveralRoutees``将会把消息发送给所有提供的routee。

路由逻辑的实现必须是线程安全的，因为它可能在actor外被使用。

路由逻辑的一个单元测试：

```scala
case class TestRoutee(n: Int) extends Routee {
  override def send(message: Any, sender: ActorRef): Unit = ()
}
 
  val logic = new RedundancyRoutingLogic(nbrCopies = 3)
 
  val routees = for (n <- 1 to 7) yield TestRoutee(n)
 
  val r1 = logic.select("msg", routees)
  r1.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(1), TestRoutee(2), TestRoutee(3)))
 
  val r2 = logic.select("msg", routees)
  r2.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(4), TestRoutee(5), TestRoutee(6)))
 
  val r3 = logic.select("msg", routees)
  r3.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(7), TestRoutee(1), TestRoutee(2)))
```

你可以停在这儿，通过``akka.routing.Router``使用 ``RedundancyRoutingLogic``，如[一个简单路由器](#simple-router-scala)中所述。

让我们继续，并使之成为一个自包含的、可配置的路由器actor。

创建一个类来扩展``Pool``，``Group``或``CustomRouterConfig``。该类是一个路由逻辑的工厂，并持有路由器的配置。在这里，我们把它变成一个``Group``。

```scala
import akka.dispatch.Dispatchers
import akka.routing.Group
import akka.routing.Router
import akka.japi.Util.immutableSeq
import com.typesafe.config.Config
 
case class RedundancyGroup(override val paths: immutable.Iterable[String], nbrCopies: Int) extends Group {
 
  def this(config: Config) = this(
    paths = immutableSeq(config.getStringList("routees.paths")),
    nbrCopies = config.getInt("nbr-copies"))
 
  override def createRouter(system: ActorSystem): Router =
    new Router(new RedundancyRoutingLogic(nbrCopies))
 
  override val routerDispatcher: String = Dispatchers.DefaultDispatcherId
}
```

这样就可以像Akka提供的路由actor完全一样使用。

```scala
for (n <- 1 to 10) system.actorOf(Props[Storage], "s" + n)
 
val paths = for (n <- 1 to 10) yield ("/user/s" + n)
val redundancy1: ActorRef =
  system.actorOf(RedundancyGroup(paths, nbrCopies = 3).props(),
    name = "redundancy1")
redundancy1 ! "important"
```

请注意我们在``RedundancyGroup``添加一个以``Config``为参数的构造函数。这样一来，我们就可能在配置中定义。

```
akka.actor.deployment {
  /redundancy2 {
    router = "docs.routing.RedundancyGroup"
    routees.paths = ["/user/s1", "/user/s2", "/user/s3"]
    nbr-copies = 5
  }
}
```

请注意``router``属性中的类的全名。路由器类必须继承``akka.routing.RouterConfig``（``Pool``，``Group``或``CustomRouterConfig``），并且有一个以``com.typesafe.config.Config``为参数的构造函数。配置的部署部分将被传递给构造函数。

```scala
val redundancy2: ActorRef = system.actorOf(FromConfig.props(),
  name = "redundancy2")
redundancy2 ! "very important"
```

###<a name="configuring-dispatchers"></a>配置调度器
创建子actor池的调度器将取自``Props``，如[调度器](04_dispatchers.md)中所述。

为了可以很容易地定义池routee的调度器，你可以在配置的部署一节中定义内联调度器。

```
akka.actor.deployment {
  /poolWithDispatcher {
    router = random-pool
    nr-of-instances = 5
    pool-dispatcher {
      fork-join-executor.parallelism-min = 5
      fork-join-executor.parallelism-max = 5
    }
  }
}
```

这是启用一个池专用调度器，你唯一需要做的。

> 注意

> 如果你使用actor群，并路由到它们的路径，然后他们将仍然使用配置在其``Props``的相同的调度器，不可能在actor创建后改变actor的调度器。

“头”路由器不能总是在相同的调度器上运行，因为它不处理同一类型的消息，因此这个特殊的actor没有使用配置的调度器，但相反，使用`RouterConfig`的``routerDispatcher``，它是actor系统默认调度器默认的。所有标准路由器允许它们在构造函数或工厂方法中设置此属性，自定义路由器必须以适当的方式实现该方法。

```scala
val router: ActorRef = system.actorOf(
  // “head” router actor will run on "router-dispatcher" dispatcher
  // Worker routees will run on "pool-dispatcher" dispatcher  
  RandomPool(5, routerDispatcher = "router-dispatcher").props(Props[Worker]),
  name = "poolWithDispatcher")
```

> 注意

> 不允许配置``routerDispatcher``为`akka.dispatch.BalancingDispatcherConfigurator`，因为用于特殊路由actor的消息不能被任意其他actor处理。

