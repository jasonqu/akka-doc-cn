# Actor引用, 路径与地址

本章描述actor如何被确定，以及在一个可能是分布式的actor系统中如何定位。这与 [Actor系统]() 的核心概念有关：固有的树形监管结构和跨多个网络节点的actor之间进行透明通讯。

.. image:: ActorPath.png

上图展示了actor系统中最重要的实体关系，请继续阅读了解详情。

###什么是Actor引用？
Actor引用是 `ActorRef` 的子类，其最重要的目的是支持向它所代表的actor发送消息。每个actor通过`self`字段来访问自己的标准（本地）引用；在给其它actor发送的消息中也缺省包含这个引用。反过来，在消息处理过程中，actor可以通过`sender()`方法来访问到当前消息的发送者的引用。

根据actor系统的配置，支持几种不同类型的actor引用：

* 纯本地actor引用，在配置为不使用网络功能的actor系统中使用。这些actor引用如果通过网络连接传给远程的JVM，将不能正常工作。
* 本地actor引用，在配置为使用远程功能的actor系统中使用，来代表同一个JVM的actor。
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
由于actor是以一种严格的树形结构样式来创建的，所以沿着子actor到父actor的监管链，一直到actor系统的根存在一条唯一的actor名字序列。这个序列可以被看做是文件系统中的文件路径，所以我们称之为“路径”。就象在一些真正的文件系统中一样，也存在所谓的“符号链接”，即一个actor也许能通过不同的路径被访问到，除了原始路径外，其它的路径都涉及到对actor实际监管祖先链的部分路径的转换方法。这些特性将在下面的内容中介绍。

一个actor路径包含一个锚点，来标识actor系统的，之后是各路径元素的连接，从根监护者到指定的actor；路径元素是路径经过的actor的名字，以"/"分隔。

#####Actor引用和路径之间有什么区别？
Actor引用标明了一个actor，其生命周期和actor的生命周期保持匹配；actor路径表示一个名称，其背后可能有也可能没有真实的actor，而且路径本身不具有生命周期，它永远不会失效。你可以创建一个actor路径，而无需创建一个actor，但你不能在创建actor引用时不创建相应的actor。

.. note::
注意
这个定义并不适用于`actorFor`，这是为什么废弃`actorFor`而选择`actorSelection`的原因之一。

你可以创建一个actor，终止它，然后创建一个具有相同路径的新actor。新创建的实例是actor的一个新的化身。它并不是一样的actor。一个指向老的化身的actor引用不适用于新的化身。发送给老的actor引用的消息不会被传递到新的化身，即使它们拥有相同的路径。

###Actor路径锚点

每一条actor路径都有一个地址组件，描述访问这个actor所需要的协议和位置，之后是从根到actor所经过的树节点上actor的名字。例如：

  "akka://my-sys/user/service-a/worker1"                   // 纯本地
  "akka.tcp://my-sys@host.example.com:5678/user/service-b" // 远程

在这里, `akka.tcp` Akka 2.2 版本默认的远程传输方式，其它的方式都是可以通过插件引入的。对使用UDO的远程主机可以使用`akka.udp`访问。对主机和端口部分的理解（即上例中的`host.example.com:5678`）决定于所使用的传输机制，但是必须遵循URI的结构标准。



