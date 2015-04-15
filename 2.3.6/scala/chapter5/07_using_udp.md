# 使用UDP

> 注：本节未经校验，如有问题欢迎提issue

UDP 无连接的数据报协议，在JDK 级别上提供两种不同的通信方式：

* 套接字可以自由地发送数据报到任何目的地，并从任何来源接收数据报
* 套接字被限定只和一个特定的远程套接字地址通信

低级API中区分是——令人困惑地——通过是否在套接字上调用`connect`方法（甚至当`connect`被调用，协议仍然是无连接的）。UDP 使用的这两种形式是使用不同的 IO 扩展提供的，如下所述。

###无连接UDP
#####简单发送

```scala
class SimpleSender(remote: InetSocketAddress) extends Actor {
  import context.system
  IO(Udp) ! Udp.SimpleSender

  def receive = {
    case Udp.SimpleSenderReady =>
      context.become(ready(sender()))
  }

  def ready(send: ActorRef): Receive = {
    case msg: String =>
      send ! Udp.Send(ByteString(msg), remote)
  }
}
```

UDP 使用最简单的形式是只发送数据报，而不需要得到回复。为此目的一个"简单的发送者"工具如上所示。UDP 扩展使用`SimpleSender`消息查询 ，由`SimpleSenderReady`通知回答。此消息的发送者是新创建的发送者actor，从此时起可以用于将数据报发送到任意的目的地 ；在此示例中，它将只发送任何收到的utf-8编码的`String`到一个预定义的远程地址。

> 注意

> 简单的发送者不会关闭本身，因为它无法知道什么时候完成工作了。当你想要关闭该发送者的短暂绑定的端口时，你需要发送一个 `PoisonPill`给它。

#####绑定（和发送）

```scala
class Listener(nextActor: ActorRef) extends Actor {
  import context.system
  IO(Udp) ! Udp.Bind(self, new InetSocketAddress("localhost", 0))

  def receive = {
    case Udp.Bound(local) =>
      context.become(ready(sender()))
  }

  def ready(socket: ActorRef): Receive = {
    case Udp.Received(data, remote) =>
      val processed = // parse data etc., e.g. using PipelineStage
      socket ! Udp.Send(data, remote) // example server echoes back
      nextActor ! processed
    case Udp.Unbind  => socket ! Udp.Unbind
    case Udp.Unbound => context.stop(self)
  }
}
```

如果你想要实现一个 UDP 服务器侦听套接字来接收传入数据报，则你需要使用`Bind`命令，如上所示。指定的本地地址可能会有一个为零的端口，此时操作系统会自动选择一个自由端口并将它分配给新的套接字。通过检查`Bound`消息可以发现实际上绑定的是哪个端口。

`Bound`消息的发送者是管理新套接字的那个actor。发送数据报是通过使用`Send`消息类型实现的，并且套接字可以通过发送一个 `Unbind` 命令来关闭，这种情况下套接字actor会回复一个`Unbound`通知。

接收的数据报发送到被`Bind`消息指定的actor，`Bound`消息将被发送给`Bind`的发送者。

###连接的 UDP
通过基于连接的 UDP API 提供的服务，类似于我们前面所述的bind-and-send服务，但主要区别是连接只是能够将发送到它连接到de  ``remoteAddress``，并将只能从该地址接收数据报。

```scala
class Connected(remote: InetSocketAddress) extends Actor {
  import context.system
  IO(UdpConnected) ! UdpConnected.Connect(self, remote)

  def receive = {
    case UdpConnected.Connected =>
      context.become(ready(sender()))
  }

  def ready(connection: ActorRef): Receive = {
    case UdpConnected.Received(data) =>
      // process data, send it on, etc.
    case msg: String =>
      connection ! UdpConnected.Send(ByteString(msg))
    case d @ UdpConnected.Disconnect => connection ! d
    case UdpConnected.Disconnected   => context.stop(self)
  }
}

```

因此在这里的例子看起来非常类似于前一个示例，最大的区别是`Send`和`Received`消息中没有远程地址信息。

> 注意

> 还有相比无连接的，使用基于连接的 UDP API有小的性能好处。如果系统启用了一个安全管理器，每个无连接的消息发送要经过一个安全检查，而在基于连接的 UDP 的情况下，安全检查连接后被缓存，因此写操作不会遭受额外的性能惩罚。








