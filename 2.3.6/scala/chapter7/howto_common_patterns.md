# 如何使用：常用模式

> 注：本节未经校验，如有问题欢迎提issue

本节列出了一些常见的actor模式，它们已被发现是有用的、优雅的或有启发意义的。所有主题都是受欢迎的，列出的示例主题包括消息路由策略，监督模式，重启处理等。作为一个特殊的奖励，这一节补充标记了贡献者的名字，如果每个Akka使用者在他或她的代码中发现重复出现的模式，并为所有人分享它的好处是多么美好的事情啊。在适和的情况下也可能加入``akka.pattern``包来创建一个[类OTP库（OTP-like library）](http://www.erlang.org/doc/man_index.html)。

### <a id="throttling_messages"></a>限制消息

贡献者：Kaspar Fischer

"一个消息节流器以确保消息不会以太高的速率发送。"

该模式详见[在Akka2中做消息限流](http://letitcrash.com/post/28901663062/throttling-messages-in-akka-2)。

### <a id="balancing_workload_across_nodes"></a>跨节点平衡负载

贡献者：Derek Wyatt

"很多时候，人们需要BalancingDispatcher的功能中包含在不同节点上拥有独立邮箱的Actor工作的规则。在这篇文章我们将探索实现这样一个概念。"

该模式详见[Akka2跨节点负载均衡](http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2>)。

### <a id="work_pulling_pattern"></a>工作拉取模式来限流和分发工作，并防止邮箱溢出

贡献者：Michael Pollmeier

"如果创建工作的速度实际上比执行它块，这种模式可以确保你的邮箱不会溢出 —— 当邮箱最终变得太满时这会导致内存溢出错误。它也让你围绕你的群集分配工作，动态地扩展规模，并且是完全无阻塞的。这是‘负载均衡模式’的一个特例。

该模式详见[工作拉取模式来限流和分发工作，并防止邮箱溢出](http://www.michaelpollmeier.com/akka-work-pulling-pattern/)。

### <a id="ordered_termination"></a>有序终止

贡献者：Derek Wyatt

"当一个actor停止时，它的子actor以未知顺序停止。子actor终止是异步的，因而是不确定的。

如果actor的孩子有顺序依赖关系，则你可能需要确保这些子actor以特定顺序关闭，从而使其postStop() 方法按正确顺序调用。"

该模式详见[一种Akka2终止器](http://letitcrash.com/post/29773618510/an-akka-2-terminator)。

### <a id="akka_amqp_proxies"></a>Akka AMQP 代理

贡献者：Fabrice Drouin

"“AMQP 代理”是将AMQP与Akka结合进行跨计算节点网络工作分发的简单方法。你将仍然编写“本地”代码，进行很少的配置，并将最终拥有一个分布式的、弹性的、容错的网格，其计算节点可以几乎以任何语言编写。"

该模式详见[Akka AMQP 代理](http://letitcrash.com/post/29988753572/akka-amqp-proxies)。

### <a id="shutdown_patterns"></a>Akka2 关闭模式

贡献者：Derek Wyatt

"当一切都结束时你如何告诉Akka关闭ActorSystem？原来竟然没有这样一个神奇的标志，没有配置设置，没有可以注册特殊回调的地方，也没有杰出的关机童话仙子用她的荣光在那完美的一刻恩典你的应用程序。她就是很普通刻薄。

在这篇文章中，我们将讨论为什么是这种情况，并为你提供一个简单的选项"在正确的时间"关闭，以及一个并不是-那么-简单-的选项来达到同样的目的。"

该模式详见[Akka2 关闭模式](http://letitcrash.com/post/30165507578/shutdown-patterns-in-akka-2)。

### <a id="distributed_graph_processing"></a>Akka分布式（内存中）图处理

贡献者：Adelbert Chang

"图在数学和计算机科学（以及其他领域）中一直是一个有趣的研究结构，而且在社交网络如Facebook和Twitter中变得更加有趣，其底层网络结构可以很好地由图来描述"。

该模式详见[Akka分布式（内存中）图处理](http://letitcrash.com/post/30257014291/distributed-in-memory-graph-processing-with-akka)。

### <a id="an_autoupdating_cache_using_actors"></a>案例研究：使用actor自动更新缓存
贡献者：Eric Pederson

"我们最近需要在一个缓慢的后端系统前构建一个高速缓存系统，并符合下列要求：

后端系统中的数据是不断更新的，所以需要每隔N分钟更新缓存。对后端系统的直接请求需要被限流。我们建立的缓存系统使用了Akka actor和 Scala 中函数作为头等对象的支持。"

该模式详见[案例研究：使用actor自动更新缓存](http://letitcrash.com/post/30509298968/case-study-an-auto-updating-cache-using-actors)。

### <a id="spider_pattern"></a>使用蜘蛛模式在actor系统中发现消息流

贡献者：Raymond Roestenburg

"构建actor系统是有趣的，可是调试它们可能很困难，你大多情况下需要在多个机器中浏览大量日志文件，来了解到底发生了什么。我敢肯定你在啃日志时会想，“嘿，这个消息跑哪了？”，“为什么这个消息引起这种效果”或“为什么这个actor永远得不到消息？”

这是蜘蛛模式的进来。"

该模式详见[使用蜘蛛模式在actor系统中发现消息流](http://letitcrash.com/post/30585282971/discovering-message-flows-in-actor-systems-with-the)。

### <a id="scheduling_periodic_messages"></a>调度周期性消息

此模式描述了如何安排周期性消息给自己，有两种不同方式。

第一种方法是在actor构造函数中设置定期消息调度，并在 ``postStop`` 中取消定时发送，否则我们可能会有多个已注册的消息发送到相同的actor。

> 注意

> 用这种方法被调度的定期消息发送将在actor重启时被重新启动。这也意味着在重新启动期间，两个tick消息之间的时间间隔可能会漂移，它基于你重新启动调度预定的消息时的时间到最后一个消息的发送时间，以及初始延迟是多长时间。最糟糕的情况是``interval``加上``initialDelay``。

```scala
class ScheduleInConstructor extends Actor {
  import context.dispatcher
  val tick =
    context.system.scheduler.schedule(500 millis, 1000 millis, self, "tick")

  override def postStop() = tick.cancel()

  def receive = {
    case "tick" =>
      // do something useful here
  }
}
```

第二种变体在actor的``preStart``方法中建立了一个初始的消息发送，然后当actor接收到此消息时设置一个新的消息发送。你还必须重写 ``postRestart``，所以我们不会调用 ``preStart`` 并重新调度一个初始消息的发送。

> 注意

> 用这种方法，即使actor负载很高，我们也不会被tick消息填满邮箱，而只会在收到前一个tick消息之后安排新tick消息发送。

```scala
class ScheduleInReceive extends Actor {
  import context._

  override def preStart() =
    system.scheduler.scheduleOnce(500 millis, self, "tick")

  // override postRestart so we don't call preStart and schedule a new message
  override def postRestart(reason: Throwable) = {}

  def receive = {
    case "tick" =>
      // send another periodic tick after the specified delay
      system.scheduler.scheduleOnce(1000 millis, self, "tick")
      // do something useful here
  }
}
```

### <a id="template_pattern"></a>模板模式
*贡献者：N. N.*

这是一种特别好的模式，因为它甚至伴随着一些空的示例代码：

```scala
class ScalaTemplate {
  println("Hello, Template!")
  // uninteresting stuff ...
}
```

> 注意

> 流传一句话：这是成名的最简单方法！

请在该文件的结尾保留这个模式。
