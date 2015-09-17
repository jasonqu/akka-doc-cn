# Actor引用, 路径与地址

本章描述actor如何被确定，以及在一个可能是分布式的actor系统中如何定位。这与 [Actor系统](02_actor_systems.md) 的核心概念有关：固有的树形监管结构和跨多个网络节点的actor之间进行透明通讯。

![](ActorPath.png)

上图展示了actor系统中最重要的实体关系，请继续阅读了解详情。

###<a name="actorref"></a>什么是Actor引用？
Actor引用是 `ActorRef` 的子类，其最重要的目的是支持向它所代表的actor发送消息。每个actor通过`self`字段来访问自己的标准（本地）引用；在给其它actor发送的消息中也缺省包含这个引用。反过来，在消息处理过程中，actor可以通过`sender()`方法来访问到当前消息的发送者的引用。

根据actor系统的配置，支持几种不同类型的actor引用：

* 纯本地actor引用，在配置为不使用网络功能的actor系统中使用。这些actor引用如果通过网络连接传给远程的JVM，将不能正常工作。
* 本地actor引用，在配置为使用远程功能的actor系统中使用，来代表同一个JVM的actor。为了能够在被发送到其它节点时仍然可达，这些引用包含了协议和远程地址信息。
* 本地actor引用的一个子类，用在路由器中（routers，即混入 了 `Router` trait的actor）。它的逻辑结构与之前的本地引用是一样的，但是向它们发送的消息会被直接重定向到它的子actor。
* 远程actor引用，代表可以通过远程通讯访问的actor，即向他们发送消息时会透明地对消息进行序列化，并发送到别的JVM。
* 有几种特殊的actor引用类型，在实际用途中比较类似本地actor引用：
  * `PromiseActorRef` 表示一个`Promise`，其目的是通过一个actor返回的响应来完成。它是由 `akka.pattern.ask` 创建的。
  * `DeadLetterActorRef`是死信服务的缺省实现，所有接收方被关闭或不存在的消息都被重新路由在此。
  * `EmptyLocalActorRef`是当查找一个不存在的本地actor路径时Akka返回的：它相当于`DeadLetterActorRef`，但是它保有其路径因此可以在网络上发送，并与其它相同路径的存活的actor引用进行比较，其中一些存活的actor引用可能在该actor消失之前被得到。
* 然后有一些内部实现，你应该永远不会用上：
  * 有一个actor引用并不表示任何actor，只是作为根actor的伪监管者存在，我们称它为“时空气泡穿梭者”。
  * 在actor创建设施启动之前运行的第一个日志服务，是一个伪actor引用，它接收日志事件并直接显示到标准输出上；它就是 `Logging.StandardOutLogger`。

###什么是Actor路径?
由于actor是以一种严格的树形结构样式来创建的，所以沿着子actor到父actor的监管链，一直到actor系统的根存在一条唯一的actor名字序列。这个序列可以被看做是文件系统中的文件路径，所以我们称之为“路径”。就像在一些真正的文件系统中一样，也存在所谓的“符号链接”，即一个actor也许能通过不同的路径被访问到，除了原始路径外，其它的路径都涉及到对actor实际监管祖先链的某部分路径进行转换的方法。这些特性将在下面的内容中介绍。

一个actor路径包含一个锚点，来标识actor系统的，之后是各路径元素的连接，从根监护者到指定的actor；路径元素是路径经过的actor的名字，以"/"分隔。

#####Actor引用和路径之间有什么区别？
Actor引用标明了一个actor，其生命周期和actor的生命周期保持匹配；actor路径表示一个名称，其背后可能有也可能没有真实的actor，而且路径本身不具有生命周期，它永远不会失效。你可以创建一个actor路径，而无需创建一个actor，但你不能在创建actor引用时不创建相应的actor。

> 注意

> 这个定义并不适用于`actorFor`，这是为什么废弃`actorFor`而选择`actorSelection`的原因之一。

你可以创建一个actor，终止它，然后创建一个具有相同路径的新actor。新创建的实例是actor的一个新的化身。它并不是一样的actor。一个指向老的化身的actor引用不适用于新的化身。发送给老的actor引用的消息不会被传递到新的化身，即使它们拥有相同的路径。

#####Actor路径锚点

每一条actor路径都有一个地址组件，描述访问这个actor所需要的协议和位置，之后是从根到actor所经过的树节点上actor的名字。例如：

    "akka://my-sys/user/service-a/worker1"                   // 纯本地
    "akka.tcp://my-sys@host.example.com:5678/user/service-b" // 远程

在这里, `akka.tcp` 是Akka 2.2及以上版本默认的远程传输方式，其它的方式都是可以通过插件引入的。对使用UDP的远程主机可以使用`akka.udp`访问。对主机和端口部分的解析（即上例中的`host.example.com:5678`）决定于所使用的传输机制，但是必须遵循URI的结构标准。

#####Actor逻辑路径
顺着actor的父监管链一直到根的唯一路径被称为actor逻辑路径。这个路径与actor的创建祖先关系完全吻合，所以当actor系统的远程调用配置（和配置中路径的地址部分）设置好后它就是完全确定的了。

#####Actor物理路径
Actor逻辑路径描述它在一个actor系统内部的功能位置，而基于配置的远程部署意味着一个actor可能在另外一台网络主机上被创建，即另一个actor系统中。在这种情况下，从根守护者穿过actor路径来找到该actor肯定需要访问网络，这是一个很昂贵的操作。因此，每一个actor同时还有一条物理路径，从actor对象实际所在的actor系统的根开始。与其它actor通信时使用物理路径作为发送方引用，能够让接收方直接回复到这个actor上，将路由延迟降到最小。

物理路径的一个重要性质是它决不会跨多个actor系统或跨JVM虚拟机。这意味着如果一个actor有祖先被远程监管，则其逻辑路径（监管树）和物理路径（actor部署）可能会分叉。


###如何获得Actor引用？
actor引用的获取方法分为两类：通过创建actor，或者通过查找actor。后一种功能又分两种：通过具体的actor路径来创建actor引用，和查询actor逻辑树。

#####创建Actor
一个actor系统通常是在根守护者上使用`ActorSystem.actorOf`创建actor来启动，然后在创建出的actor中使用`ActorContext.actorOf`来展开actor树。这些方法返回的是指向新创建的actor的引用。每个actor都拥有到它的父亲，它自己和它的子actor的引用（通过`ActorContext`访问）。这些引用可以与消息一起被发送给别的actor，以便接收方直接回复。

#####通过具体的路径来查找actor

另外，可以使用`ActorSystem.actorSelection`来查找actor引用。“选择”可在已有actor与被选择的actor进行通讯的时候用到，在投递每条消息的时候都会用到查找。

为了获得一个绑定到指定actor生命周期的`ActorRef`，你需要发送一个消息，如内置的`Identify`信息，向指定的actor，所获得的`sender()`即为所求。

> 注意

> `actorFor`因被`actorSelection`替代而废弃，因为`actorFor`对本地和远程的actor表现有所不同。对一个本地actor引用，被查找的actor需要在查找之前就存在，否则获得的引用是一个`EmptyLocalActorRef`。即使后来与实际路径相符的actor被创建，所获得引用仍然是这样。对于`actorFor`行为获得的远程actor
引用则不同，每条消息的发送都会在远程系统中进行一次按路径的查找。

###绝对路径 vs 相对路径
除了`ActorSystem.actorSelection`还有一个`ActorContext.actorSelection`，这是可以在任何一个actor实例中通过`context.actorSelection`访问的。它的actor查找与`ActorSystem`的返回值非常类似，不同在于它的路径查找是从当前actor开始的，而不是从actor树的根开始。可以用 ".." 路径来访问父actor. 例如，你可以向一个指定兄弟发送消息：

	context.actorSelection("../brother") ! msg

当然绝对路径也可以在 context 中使用，即

	context.actorSelection("/user/serviceA") ! msg

也能正确运行。


#####查询逻辑Actor树
由于actor系统是一个类似文件系统的树形结构，对actor路径的匹配与Unix shell中支持的一样：你可以将路径（中的一部分）用通配符(«*» 和 «?»)替换，来组成对0个或多个实际actor的选择。由于匹配的结果不是一个单一的actor引用，它拥有一个不同的类型`ActorSelection`，这个类型不完全支持`ActorRef`的所有操作。选择也可以用`ActorSystem.actorSelection`或`ActorContext.actorSelection`两种方式来获得，并且支持发送消息：

	context.actorSelection("../*") ! msg

会将msg发送给包括当前actor在内的所有兄弟。对于用 `actorFor`获取的actor引用，为了进行消息的发送，会对监管树进行遍历。由于在消息到达其接收者的过程中，与查询条件匹配的actor集合可能会发生变化，要监视查询的实时变化是不可能的。如果要做这件事情，通过发送一个请求，收集所有的响应来解决不确定性，提取所有的发送方引用，然后监视所有被发现的具体actor。这种处理actor选择的方式也许会在未来的版本中进行改进。

#####总结: `actorOf` vs. `actorSelection`vs. `actorFor`

> Note

> 以上部分所描述的细节可以简要地总结和记忆成：
* `actorOf` 永远都只会创建一个新的actor，这个新的actor是actorOf所调用上下文（可以是任意一个actor或actor系统本身）的直接子actor
* `actorSelection`只会在消息送达后查找已经存在的actor集合，即不会创建actor，也不会在创建选择集合时验证actor是否存在。
* `actorFor`（废弃，已经被`actorSelection`取代） 永远都只是查找到一个已存在的actor，不会创建新的actor。


###Actor引用和路径相等性

`ActorRef`的相等性与`ActorRef`的目的匹配，即一个`ActorRef`对应一个目标actor化身。两个actor引用进行比较时，如果它们有相同的路径且指向同一个actor化身，则两者相等。指向一个已终止的actor的引用，与指向具有相同路径但却是另一个（重新创建）actor的引用是不相等的。需要注意的是，由于失败造导致的actor重启，仍意味着它是同一个actor化身，即重新启动对`ActorRef`消费者是不可见的。

由`actorFor`获得的远程actor引用不包括其身份的所有信息，因此，这种引用不能等于`actorOf`，`sender`或`context.self`的引用。正因如此`actorFor`被`actorSelection`替换废弃。

如果你需要跟踪一个集合中的actor引用，并不关心具体的actor化身，你可以使用`ActorPath`为键（key），因为目标actor的标识符在比较actor路径时没有被用到。


###重用Actor路径
当一个actor被终止，其引用将指向一个死信邮箱，DeathWatch将发布其最终的转变，并且一般地它也不会起死回生（因为actor的生命周期不允许这样）。虽然以后可能创建一个具有相同路径的actor——如果无法保留actor系统开始以来创建的所有可用actor，则无法保证其反向成立——但是这不是一个好的实践：通过`acterFor`获取的已经‘死亡’的远程actor引用突然再次开始工作，但没有这种过渡和任何其他事件之间顺序的任何保证，因此，该路径的新居民可能收到本意是送给其以前住户的消息。

在某些非常特殊的情况下这可能是正确的事情，但一定要限制这种处理只能由其监管者操作，因为它是唯一可以可靠地检测名称正确注销的actor，在注销之前的新创建子actor的操作将失败。

它在测试中可能也是必要的，当测试对象取决于某个特定路径被实例化的时候。在这种情况下，最好mock其监管者，这样它会将终止消息转发至测试过程正确的点，使后者能够等待登记名字的正确注销。

###与远程部署之间的互操作
当一个actor创建一个子actor，actor系统的部署者会决定新的actor是在同一个jvm中还是在其它节点上。如果是后者，actor的创建会通过网络连接引到另一个jvm中进行，因而在另一个actor系统中。远程系统会将新的actor放在一个专为这种场景所保留的特殊路径下，新的actor的监管者将会是一个远程actor引用（代表触发它创建动作的actor）。这时，`context.parent`（监管者引用）和`context.path.parent`（actor路径上的父actor）表示的actor是不同的。然而，在其监管者中查找这个actor的名称将会在远程节点上找到它，保持其逻辑结构，例如向另一个未确定(unresolved)的actor引用发送消息。

![](RemoteDeployment.png)

###路径中的地址部分用来做什么？
在网络上传送actor引用时，是用它的路径来表示的。因此，它的路径必须包括能够用来向它所代表的actor发送消息的完整信息。这一点是通过将协议、主机名和端口编码在路径字符串的地址部分做到的。当actor系统从远程节点接收到一个actor路径，会检查它的地址部分是否与自己的地址相同，如果相同，那么会将这条路径解析为本地actor引用，否则解析为一个远程actor引用。


###<a name="toplevel-paths"></a>Actor路径的顶级作用域
在路径树的根上是根监管者，所有其他actor都可以从通过它找到；它的名字是`"/"`。在第二个层次上是以下这些：

* `"/user"` 是所有由用户创建的顶级actor的监管者；用 `ActorSystem.actorOf`创建的actor在其下。
* `"/system"` 是所有由系统创建的顶级actor的监管者，如日志监听器，或由配置指定在actor系统启动时自动部署的actor。
* `"/deadLetters"` 是死信actor，所有发往已经终止或不存在的actor的消息会被重定向到这里（以尽最大努力为基础：即使在本地JVM，消息也可能丢失）
* `"/temp"`是所有系统创建的短时actor的监管者，例如那些在`ActorRef.ask`的实现中用到的actor。
* `"/remote"` 是一个人造虚拟路径，用来存放所有其监管者是远程actor引用的actor。

需要为actor构建这样的名称空间源于一个核心的非常简单的设计目标：在树形结构中的一切都是一个actor，以及所有的actor都以相同方式工作。因此，你不仅可以查找你所创建的actor，你也可以查找系统守护者并发送消息（在这种情况下它会忠实地丢弃之）。这个强大的原则意味着不需要记住额外的怪异模式，它使整个系统更加统一和一致。

如果您想了解更多关于actor系统的顶层结构，参考[顶级监管者](04_supervision_and_monitoring.md#toplevel-supervisors)。

