# 使用TCP

> 注：本节未经校验，如有问题欢迎提issue

贯穿本节的代码片段假定以下imports：

```scala
import akka.actor.{ Actor, ActorRef, Props }
import akka.io.{ IO, Tcp }
import akka.util.ByteString
import java.net.InetSocketAddress
```

所有的Akka I/O API 都通过管理器对象来访问。当使用I/O API时，第一步是获得适当的管理器对象的引用。下面的代码演示如何获取``Tcp``管理器的引用。

```scala
import akka.io.{ IO, Tcp }
import context.system // implicitly used by IO(Tcp)

val manager = IO(Tcp)
```

管理器是一个actor，来处理底层低级别 I/O 资源(selectors, channels)并为特定任务，如监听传入的连接，来实例化工作者。

###连接

```scala
object Client {
  def props(remote: InetSocketAddress, replies: ActorRef) =
    Props(classOf[Client], remote, replies)
}

class Client(remote: InetSocketAddress, listener: ActorRef) extends Actor {

  import Tcp._
  import context.system

  IO(Tcp) ! Connect(remote)

  def receive = {
    case CommandFailed(_: Connect) =>
      listener ! "connect failed"
      context stop self

    case c @ Connected(remote, local) =>
      listener ! c
      val connection = sender()
      connection ! Register(self)
      context become {
        case data: ByteString =>
          connection ! Write(data)
        case CommandFailed(w: Write) =>
          // O/S buffer was full
          listener ! "write failed"
        case Received(data) =>
          listener ! data
        case "close" =>
          connection ! Close
        case _: ConnectionClosed =>
          listener ! "connection closed"
          context stop self
      }
  }
}
```

连接到一个远程地址的第一步是将`Connect`消息发送到 TCP 管理器 ；除了上面所示的简单形式外，还可以指定要绑定的本地``InetSocketAddress``和要应用的套接字选项列表。

> 注意

> SO_NODELAY (Windows下是TCP_NODELAY）套接字选项在Akka中默认为 true，独立于操作系统的默认设置。此设置禁用Nagle算法，大大提高对于大多数应用程序的延迟。此设置可以通过在``Connect``消息的套接字选项的列表中加入``SO.TcpNoDelay(false)``来重写。

TCP 管理器然后将要么回复`CommandFailed`，要么产生一个内部的actor代表新的连接。这位新actor然后将向`Connect`消息的原始发送人发送一个`Connected`消息。

为了激活新的连接，`Register`消息必须发送到连接actor，通知它谁将从套接字接收数据。此步骤完成之前不能使用连接，而且还有一个内部超时，在此之后如果没有收到`Register`消息，连接actor将关闭自身。

连接actor观察注册的处理程序，并在处理者终止时关闭连接，从而清理与该连接相关的所有内部资源。

上面的示例中actor使用`become`来从无连接状态切换到已连接状态，演示那种状态下被观察到的命令和事件。关于`CommandFailed`参见下文的讨论——[读写限流](#throttling-reads-and-writes)。`ConnectionClosed`是一个特质，它标志着不同的连接关闭事件。最后一行中以同样的方式处理所有的连接关闭事件。有可能监听更多的细粒度的连接关闭事件，请参阅下面的[关闭连接](#closing-connections)。

###接受连接

```scala
class Server extends Actor {

  import Tcp._
  import context.system

  IO(Tcp) ! Bind(self, new InetSocketAddress("localhost", 0))

  def receive = {
    case b @ Bound(localAddress) =>
      // do some logging or setup ...

    case CommandFailed(_: Bind) => context stop self

    case c @ Connected(remote, local) =>
      val handler = context.actorOf(Props[SimplisticHandler])
      val connection = sender()
      connection ! Register(handler)
  }

}
```

要创建一个 TCP 服务器并监听入站连接，需要发送一个``Bind``命令到 TCP 管理器。这将指示 TCP 管理器来侦听一个特定`InetSocketAddress`的 TCP 连接；可将端口指定为 0来绑定一个随机端口。

发送`Bind`消息的actor将收到`Bound`消息，表示服务器已准备好接收传入的连接 ；此消息还包含实际绑定到套接字的`InetSocketAddress`（即解析号的 IP 地址和正确的端口号）。

从这个点开始，处理连接的过程与传出连接是相同的。该示例演示了，当发送`Register`消息时，来自某个连接的读取可以委派给另一位被指定处理程序的actor。写操作可以从系统中任何一个actor发送到连接actor （即发送`Connected`消息的actor）。简单化的处理程序定义如下：

```scala
class SimplisticHandler extends Actor {
  import Tcp._
  def receive = {
    case Received(data) => sender() ! Write(data)
    case PeerClosed     => context stop self
  }
}
```

在发送时考虑到失败可能性的更完整示例，请参阅下面的[读写限流](#throttling-reads-and-writes)。

对传出连接的唯一区别是内部actor管理监听端口 —— `Bound`消息的发件人 —— 观察在`Bind`消息中被指定为`Connected`消息接收者的那个actor。那个actor终止时监听端口将会封闭，与它相关的所有资源将都被释放 ；在这一点上，现有连接不会被终止。

###关闭连接
可以通过发送``Close``, ``ConfirmedClose``或``Abort``之一的命令到连接actor关闭连接。

通过发送``FIN``消息，``Close``将关闭该连接，但没有等待远端的确认。挂起的写操作将被执行(flush)。如果关闭成功, 监听器将收到``Closed``通知。

通过发送``FIN``消息，``ConfirmedClose``将关闭发送方的连接，但仍会收到数据直到远端也关闭连接。挂起的写操作将被执行(flush)。如果关闭成功, 监听器将收到``ConfirmedClosed``通知。

通过发送``RST``消息到远端，``Abort``将立即终止连接。挂起的写操作将不会被执行(flush)。如果关闭成功, 监听器将收到``Aborted``通知。

如果该连接被远端关闭，监听器将收到``PeerClosed``通知。默认情况下，该连接然后也会被本端自动关闭。要支持半闭连接，请将``Register``消息的``keepOpenOnPeerClosed``成员设置为 ``true``，在此情况下连接将保持打开状态，直到它接收到一个上文介绍的关闭命令。

每当一个错误发生并迫使该连接关闭时，``ErrorClosed``将发送到监听器。

所有关闭通知都是``ConnectionClosed``的子类型，所以不需要细粒度事件的监听器可以以相同的方式处理所有的关闭事件。

###写入一个连接
一旦连接建立，数据可以从任何actor通过 ``Tcp.WriteCommand`` 的形式发送给连接。``Tcp.WriteCommand`` 是一个抽象类，有三个具体的实现：

* Tcp.Write

  最简单的 ``WriteCommand`` 实现，它包装了一个 ``ByteString`` 实例和一个"ack"事件。一个``ByteString`` （如在[这一节](05_io.md#bytestring_scala)解释的） 建模了一个或多个不可变的内存数据块，最大可达2GB (2^31个字节）。

* Tcp.WriteFile

  如果你想要从一个文件发送"原始"数据，你可以高效地使用``Tcp.WriteFile``命令。它将允许你指定磁盘上一块（连续的）字节在连接中发送，而无需首先将其加载到JVM内存中。``Tcp.WriteFile``可以"hold"大于2 GB的数据和并根据需要提供"ack"事件。

* Tcp.CompoundWrite

  有时你可能想要组（或交错）若干 ``Tcp.Write``和/或 ``Tcp.WriteFile``命令到一个原子写命令，一次写入连接。``Tcp.CompoundWrite``允许你做到这一点，并提供三个优点：

  1. 如下节中所述，TCP连接actor一次只能处理一个单一写命令。通过组合几个写到一个``CompoundWrite``，你可以让他们通过连接发送，并拥有最低开销，也无需像"一勺勺喂"一样通过*基于ACK*的消息协议发送给连接actor。
  2. 因为``WriteCommand``是原子的，你可以确定没有其他actor可以在你的写序列中"注入"其他写操作，如果你将它们组合为一个单一的 ``CompoundWrite``。在多个actor写入相同连接的情况下，这可以是一个重要的特性，通过别的方式很难实现。
  3. ``CompoundWrite``的"子写操作"是普通的``Write``或``WriteFile``命令，其本身可以请求"ack"事件。这些ACK会在相应的"子写操作"被完成时尽快发送。这允许你将附加多个ACK到``Write``或 ``WriteFile``（通过组合一个要求ACK的空写）或让连接actor通过在任意点发送中间ACK确认``CompoundWrite``的传输进度。

###<a name="throttling-reads-and-writes"></a>读写限流
TCP 连接actor的基本模型是它有没有内部缓冲 （即它一次只能处理一个写操作，意味着它可以缓冲一次写入，直到写入全部被传递到操作系统内核）。写入和读取的拥塞情况需要在用户级别上处理。

高压写有三种操作模式

* *基于ACK*：每个`Write`命令装载一个任意的对象，并且如果该对象不是``Tcp.NoAck``，则它将在所有包含数据成功写入到套接字后返回给`Write`的发送人。如果收到此确认之前没有启动其他写操作，则由于缓冲区重用没有失败可以发生。
* *基于NACK*：在前一个写操作完成之前，每一个新到达的写都将被回复一个包含失败写操作的`CommandFailed`消息。仅仅依靠这种机制要求实现协议容忍跳过写操作 (例如如果每个写入操作自身是一个有效的消息，并且不要求所有都送达）。通过在连接激活阶段的`Register`消息中设置``useResumeWriting``标志为 ``false``来启用此模式。
* *基于NACK与写挂起*： 这种模式非常类似于基于NACK的模式，但一旦一次写操作失败，则没有进一步写操作会成功直到接收到`ResumeWriting`消息。一旦上一次接受的写入操作完成，将由一个`WritingResumed`消息回答此消息。如果驱动连接的actor实现缓冲，并在等到`WritingResumed`信号后重发NACK标记的消息，则每个消息确切地只传递一次给网络套接字。

这些高压写模型在下面的示例中进行了完整演示 （除了相当专业化的第二个）。完整的相关资源可以在[github](http://github.com/akka/akka/tree/v2.3.6/akka-docs/rst/scala/code/docs/io/EchoServer.scala)上获得。

高压读有两种操作模式

* *推送读*： 在此模式下连接actor一旦收到`Received`事件，就向注册的读actor发送传入的数据。每当读actor想要通知到远程 TCP 端点压力过大时，它可以发送 ``SuspendReading``消息到连接actor表示它想要暂停接收新数据。没有收到`Received`事件直到`ResumeReading`被发送，表示接收actor又准备好了。
* *拉取读*： 发送一个`Received`事件后连接actor会自动挂起从套接字接受数据，直到读actor发送一个`ResumeReading`信息，作为信号表示它准备好要处理更多的输入数据。因此新的数据是通过发送`ResumeReading`消息从连接"拉"到的。

> 注意

> 显而易见，所有这些流控制方案只能在一对写者/读者和一个连接actor之间工作；一旦多个actor发送写命令到一个单独的连接，可以会出现不一致的结果。

###基于ACK的高压写
为下面示例能恰当的工作，很重要的一点是当远端关闭其写入时，要配置连接保持半打开状态： 这允许 ``EchoHandler``在连接完全关闭之前将所有未提交数据写到客户端。它是使用连接激活的标志启用的 （观察`Register`消息）：

```scala
case Connected(remote, local) =>
  log.info("received connection from {}", remote)
  val handler = context.actorOf(Props(handlerClass, sender(), remote))
  sender() ! Register(handler, keepOpenOnPeerClosed = true)
```

有了这样的让我们深入看下处理程序：

```scala
  // storage omitted ...
class SimpleEchoHandler(connection: ActorRef, remote: InetSocketAddress)
  extends Actor with ActorLogging {

  import Tcp._

  // sign death pact: this actor terminates when connection breaks
  context watch connection

  case object Ack extends Event

  def receive = {
    case Received(data) =>
      buffer(data)
      connection ! Write(data, Ack)

      context.become({
        case Received(data) => buffer(data)
        case Ack            => acknowledge()
        case PeerClosed     => closing = true
      }, discardOld = false)

    case PeerClosed => context stop self
  }

  // storage omitted ...
}
```

原理很简单： 当写如一大块数据后，总是在发送下一个块之前等待``Ack``发送回来。在等待期间，我们切换行为来缓冲新传入的数据。所使用的辅助函数有点长但并不复杂：

```scala
private def buffer(data: ByteString): Unit = {
  storage :+= data
  stored += data.size

  if (stored > maxStored) {
    log.warning(s"drop connection to [$remote] (buffer overrun)")
    context stop self

  } else if (stored > highWatermark) {
    log.debug(s"suspending reading")
    connection ! SuspendReading
    suspended = true
  }
}

private def acknowledge(): Unit = {
  require(storage.nonEmpty, "storage was empty")

  val size = storage(0).size
  stored -= size
  transferred += size

  storage = storage drop 1

  if (suspended && stored < lowWatermark) {
    log.debug("resuming reading")
    connection ! ResumeReading
    suspended = false
  }

  if (storage.isEmpty) {
    if (closing) context stop self
    else context.unbecome()
  } else connection ! Write(storage(0), Ack)
}
```

最有趣的部分也许是最后一行： 一个``Ack``从缓冲区中删除最旧的数据块，如果这是最后一个块则我们要么关闭连接 （如果同行已经关闭了它的一半）要么返回到空闲的行为；不然我们就发送下一个缓冲块，并等待下一个``Ack``。

高压也可以传播，穿过读侧回到连接另一端的写侧，通过将`SuspendReading`命令发送到连接actor。这将导致不再从套接字读取任何数据 （尽管这会延迟发生，因为它需要一些时间，直到连接actor处理这个命令，因此适当的头部缓冲区应该出现) ，反过来这将导致我们这一端的操作系统内核缓冲区填满，然后 TCP 窗口机制将停止远端写，填满其写入缓冲区，直到最后另一端的写者无法推如任何数据到套接字中了。这是端到端高压如何在跨 TCP 连接实现的。

###带有挂起的基于NACK高压写

```scala
class EchoHandler(connection: ActorRef, remote: InetSocketAddress)
  extends Actor with ActorLogging {

  import Tcp._

  case class Ack(offset: Int) extends Event

  // sign death pact: this actor terminates when connection breaks
  context watch connection

  // start out in optimistic write-through mode
  def receive = writing

  def writing: Receive = {
    case Received(data) =>
      connection ! Write(data, Ack(currentOffset))
      buffer(data)

    case Ack(ack) =>
      acknowledge(ack)

    case CommandFailed(Write(_, Ack(ack))) =>
      connection ! ResumeWriting
      context become buffering(ack)

    case PeerClosed =>
      if (storage.isEmpty) context stop self
      else context become closing
  }

  // buffering ...

  // closing ...

  override def postStop(): Unit = {
    log.info(s"transferred $transferred bytes from/to [$remote]")
  }

  // storage omitted ...
}
  // storage omitted ...
```

这里的原则是保持写，直到接收到一个`CommandFailed`，仅使用确认来修剪重发缓冲区。当收到了此类故障时，过渡到一个不同的处理状态，并处理重发队列中的所有数据：

```scala
def buffering(nack: Int): Receive = {
  var toAck = 10
  var peerClosed = false

  {
    case Received(data)         => buffer(data)
    case WritingResumed         => writeFirst()
    case PeerClosed             => peerClosed = true
    case Ack(ack) if ack < nack => acknowledge(ack)
    case Ack(ack) =>
      acknowledge(ack)
      if (storage.nonEmpty) {
        if (toAck > 0) {
          // stay in ACK-based mode for a while
          writeFirst()
          toAck -= 1
        } else {
          // then return to NACK-based again
          writeAll()
          context become (if (peerClosed) closing else writing)
        }
      } else if (peerClosed) context stop self
      else context become writing
  }
}
```

应指出的是当前缓冲的所有写操作，在进入这种状态时也都已被发送到连接actor，这意味着`ResumeWriting`消息是在那些写操作之后入队的，导致接收所有突出的`CommandFailed`消息 （在这种状态被忽略）在收到`WritingResumed`信号之前。后一种消息只在内部排队的写操作完全完成时，由连接actor发送，意味着一个后续的写操作不会失败。这被`EchoHandler`用来为头十个写操作切换到基于ACK的方式，在一次失败后和恢复到乐观完全写行为之前。

```scala
def closing: Receive = {
  case CommandFailed(_: Write) =>
    connection ! ResumeWriting
    context.become({

      case WritingResumed =>
        writeAll()
        context.unbecome()

      case ack: Int => acknowledge(ack)

    }, discardOld = false)

  case Ack(ack) =>
    acknowledge(ack)
    if (storage.isEmpty) context stop self
}
```

仍在发送所有数据时关闭连接，相比基于ACK的办法涉及更多内容：这个想法是，总是发送所有未完成的消息和确认所有成功的写操作，并且如果发生故障则切换行为等待`WritingResumed`事件并重新开始。

辅助函数非常类似于基于 ACK 的例子：

```scala
private def buffer(data: ByteString): Unit = {
  storage :+= data
  stored += data.size

  if (stored > maxStored) {
    log.warning(s"drop connection to [$remote] (buffer overrun)")
    context stop self

  } else if (stored > highWatermark) {
    log.debug(s"suspending reading at $currentOffset")
    connection ! SuspendReading
    suspended = true
  }
}

private def acknowledge(ack: Int): Unit = {
  require(ack == storageOffset, s"received ack $ack at $storageOffset")
  require(storage.nonEmpty, s"storage was empty at ack $ack")

  val size = storage(0).size
  stored -= size
  transferred += size

  storageOffset += 1
  storage = storage drop 1

  if (suspended && stored < lowWatermark) {
    log.debug("resuming reading")
    connection ! ResumeReading
    suspended = false
  }
}
```

###带有拉取模式高压写
当使用拉取读时，来自套接字的数据在可用时会被尽快发送到actor。在前面的Echo服务器示例中的情况下，这意味着我们需要维护一个传入数据缓冲区来保持它，因为写入的速度可能会低于新数据到达的速度。

在拉取模式下这个缓冲区可以完全消除，如下面的代码段所示：

```scala
override def preStart: Unit = connection ! ResumeReading

def receive = {
  case Received(data) => connection ! Write(data, Ack)
  case Ack            => connection ! ResumeReading
}
```

这里的想法是，直到以前写操作已被连接actor完全确认才恢复读。每个拉取模式连接actor都是从挂起状态开始。要启动数据流我们发送一个 ``ResumeReading`` 到``preStart``方法告诉连接actor我们准备好接收第一个块数据了。因为我们只在先前的数据块已经被完全写入的情况下才恢复读，所以就没有必要维持一个缓冲区。

要对一个出站连接启用拉取读,`Connect`的 ``pullMode`` 参数应被设置为 ``true``：

```scala
IO(Tcp) ! Connect(listenAddress, pullMode = true)
```

#####入站连接的拉取读模式
前一节演示了如何为出站连接启用拉取读模式，但也可能创建一个监听器actor具有这种读模式，通过设置 ``Bind`` 命令的 ``pullMode`` 参数设置为 ``true``：

```scala
IO(Tcp) ! Bind(self, new InetSocketAddress("localhost", 0), pullMode = true)
```

此设置的影响之一是此监听器actor所接受的所有连接将都使用拉取读模式。

此设置的另一个影响是除了所有的入站连接被设置为拉取读模式，接受连接也变为基于拉取的了。这意味着在处理一个（或多个） `Connected`事件后监听器actor一定要通过发送它一条 `ResumeAccepting` 消息来恢复。

拉取模式中的侦听器actor以挂起方式开始来接受连接，在绑定成功后必须发送`ResumeAccepting`命令到监听actor：

```scala
case Bound(localAddress) =>
  // Accept connections one by one
  sender ! ResumeAccepting(batchSize = 1)
  context.become(listening(sender))
```

在处理传入的连接后，我们需要再一次恢复接收：

```scala
def listening(listener: ActorRef): Receive = {
  case Connected(remote, local) =>
    val handler = context.actorOf(Props(classOf[PullEcho], sender))
    sender ! Register(handler, keepOpenOnPeerClosed = true)
    listener ! ResumeAccepting(batchSize = 1)
}
```

`ResumeAccepting` 接受一个 ``batchSize`` 参数，指定在需要下一个 `ResumeAccepting` 消息恢复处理新的连接之前，接受多少新的连接。



