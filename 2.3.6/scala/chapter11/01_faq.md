# 常见问题

### Akka Project

##### Where does the name Akka come from?

It is the name of a beautiful Swedish [mountain](https://lh4.googleusercontent.com/-z28mTALX90E/UCOsd249TdI/AAAAAAAAAB0/zGyNNZla-zY/w442-h331/akka-beautiful-panorama.jpg)
up in the northern part of Sweden called Laponia. The mountain is also sometimes
called 'The Queen of Laponia'.

Akka is also the name of a goddess in the Sámi (the native Swedish population)
mythology. She is the goddess that stands for all the beauty and good in the
world. The mountain can be seen as the symbol of this goddess.

Also, the name AKKA is the a palindrome of letters A and K as in Actor Kernel.

Akka is also:

* the name of the goose that Nils traveled across Sweden on in [The Wonderful Adventures of Nils](http://en.wikipedia.org/wiki/The_Wonderful_Adventures_of_Nils) by the Swedish writer Selma Lagerlöf.
* the Finnish word for 'nasty elderly woman' and the word for 'elder sister' in the Indian languages Tamil, Telugu, Kannada and Marathi.
* a [font](http://www.dafont.com/akka.font)
* a town in Morocco
* a near-earth asteroid

### Actor总体
##### 当我在Actor中使用Future时，sender()/getSender()消失了，为什么呢？
当在Actor内部使用Future回调时需要小心，避免闭合包含Actor的引用，即对闭合的Actor不在回调内调用方法或访问其内的可变状态。这将打破Actor封装，因为回调将被调度与闭合的Actor并发执行，所以可能会引入同步 bug 和竞态条件。不幸的是没有编译时的方法发现这些非法访问。

更多内容参见[Actor与共享可变状态](../chapter2/07_akka_and_the_java_memory_model.md#jmm-shared-state)的文档。

##### 为什么发生OutOfMemoryError？
产生OutOfMemoryError错误的原因很多。例如，在基于纯推送系统中，消费者的消息处理速度可能低于相应消息生产者，必须添加某种消息流量控制。否则邮件将在消费者的邮箱中排队，从而填满堆内存。

一些寻找灵感的文章：

* [Balancing Workload across Nodes with Akka 2](http://letitcrash.com/post/29044669086/balancing-workload-across-nodes-with-akka-2)
* [Work Pulling Pattern to prevent mailbox overflow, throttle and distribute work](http://www.michaelpollmeier.com/akka-work-pulling-pattern/)

### Actors Scala API

##### 我怎么会得到`receive`方法丢失消息的编译时错误呢？

一种解决方案来帮助你获得警告，因为基于你的actor输入输出消息实现基本特质的定义，编译时将检查的处理消息的全面性，如有遗漏将会告警。

这里有一个例子，编译器将发出警告你接收的比配并不是详尽无遗的：

```scala
object MyActor {
  // these are the messages we accept
  sealed abstract trait Message
  case class FooMessage(foo: String) extends Message
  case class BarMessage(bar: Int) extends Message

  // these are the replies we send
  sealed abstract trait Reply
  case class BazMessage(baz: String) extends Reply
}

class MyActor extends Actor {
  import MyActor._
  def receive = {
    case message: Message => message match {
      case BarMessage(bar) => sender ! BazMessage("Got " + bar)
      // warning here:
      // "match may not be exhaustive. It would fail on the following input: FooMessage(_)"
    }
  }
}
```

### Remoting

##### 我想发送到远程系统，但它没有做任何事情

请确保你有在两端上都启用远程处理：客户端和服务器。两端都需要配置主机名和端口，而且你将需要知道服务器端口；客户端在大多数情况下可以使用一个自动的港口（即配置端口为零）。如果这两个系统在同一网络主机上运行，它们的端口必须是不同的。

如果仍然看不到任何东西，看看远程的生命周期事件的日志记录了什么 （通常在 INFO 级别） 或打开[辅助的远程日志记录选项](../chapter6/02_logging.md#logging-remote-java)，以查看所有发送和接收消息 （在DEBUG级别）。

##### 调试远程处理问题时，应启用哪些选项？
详见[远程配置](../chapter5/03_remoting.md#remote-configuration-scala)，典型的候选人是：

* `akka.remote.log-sent-messages`
* `akka.remote.log-received-messages`
* `akka.remote.log-remote-lifecycle-events` (this also includes deserialization errors)

##### 远程Actor的名字是什么？
当你想要将消息发送到远程主机上的Actor时，你需要知道它的[完整路径](../chapter2/05_actor_references_paths_and_addresses.md#addressing)，格式如下：

    akka.protocol://system@host:1234/user/my/actor/hierarchy/path

所需要的部分是：

* ``protocol``是要用来与远程系统进行通信的协议。大多数情况下是 `tcp`。

* ``system``是远程系统的名称 （必须完全匹配，区分大小写!)

* ``host``是远程系统的 IP 地址或 DNS 名称，它必须匹配该系统的配置 （即 `akka.remote.netty.hostname`）

* ``1234`` 是侦听连接和接收消息的远程系统的端口号

* ``/user/my/actor/hierarchy/path`` 是远程Actor在远程系统监督层次结构中的绝对路径 ，包括系统的监护者(即 ``/user``还有其他如``/system``承载日志记录器，``/temp`` 保存使用``ask()``创建的临时Actor引用，`/remote`启用远程部署等);这符合该Actor在远程主机上打印它自身的引用的方式，例如在日志输出。

##### 为什么没有收到远程Actor的答复？
最常见的原因是本地系统名称 （即上述答案中的 ``system@host:1234`` 部分） 是从远程系统的网络位置不可以到达的，例如因为``host``被配置为为``0.0.0.0``，``localhost``或 NAT'ed IP 地址。

##### 消息如何可靠传递？
一般的规则是**至多一次交付(at-most-once delivery)**，即没有保证。可以在其上面建立更强的可靠性，Akka提供了工具来这样做。

详见[消息传递可靠性](../chapter2/08_message_delivery_reliability.md#message-delivery-reliability)。

### 调试

##### 如何打开调试日志记录
若要打开actor系统调试日志记录，添加如下配置：

    akka.loglevel = DEBUG

若要打开不同类型的调试日志记录，添加如下配置：

* ``akka.actor.debug.receive`` 将记录发送到一个actor的所有消息，如果actor `receive`方法是 ``LoggingReceive``
* ``akka.actor.debug.autoreceive`` 将记录发送给所有actor的所有*特殊*消息如``Kill``， ``PoisonPill``等
* ``akka.actor.debug.lifecycle`` 将记录所有actor的所有actor生命周期事件

更多内容参见[日志记录](../chapter6/02_logging.md#logging-scala) 和 [追踪Actor调用](../chapter3/09_testing_actor_systems.md#actor-logging-scala) 。