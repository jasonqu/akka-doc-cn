# ZeroMQ

> 注： 直接从Akka 2.0文档中复制，没有仔细检查

Akka提供一个 ZeroMQ 模块对 ZeroMQ 连接进行抽象从而允许Akka actor之间在ZeroMQ连接之上进行消息交互。这些消息可以是专有格式或者使用Protobuf来定义。socket actor缺省就具有容错性，当你调用newSocket方法创建新的Socket时它会恰当地对socket进行重新初始化。

ZeroMQ 在多线程方面有较大的强制性，所以配置选项 `akka.zeromq.socket-dispatcher` 一定要设置成 PinnedDispatcher, 这是因为实际的ZeroMQ socket只能在创建它的线程中访问。

Akka的ZeroMQ模块是按照JZMQ中的API编写的，JZMQ使用JNI与本地ZeroMQ库通信，但Akka ZeroMQ模块没有用JZMQ，用的是ZeroMQ的Scala绑定，与本地ZeroMQ库通信是通过JNA完成的. 换句话说，这个模块所需要的本地库只有本地 ZeroMQ 库. 使用scala库的好处是你不需要编译和管理本地的信赖，当然这会有一些性能上的损失。Scala绑定与JNI绑定是兼容的，如果你的确需要获得最好的性能，直接用后者替换前者即可。

> 注意

> ``zeromq-scala-bindings``当前所用的版本只兼容 zeromq 2 ；zeromq 3 不受支持。

###连接

ZeroMQ 支持多种连接模式, 每一种用来满足不同的需求。 目前这个模块支持发布－订阅型连接和基于Router－Dealer的连接. 为了发起连接或接收连接，必须创建一个socket. Socket都是用 akka.zeromq.ZeroMQExtension创建的, 例如:

```scala
import akka.zeromq.ZeroMQExtension
val pubSocket = ZeroMQExtension(system).newSocket(SocketType.Pub,
  Bind("tcp://127.0.0.1:21231"))
```

或者通过导入 akka.zeromq._ 包来获得隐式的newSocket方法.

```scala
import akka.zeromq._
val pubSocket2 = system.newSocket(SocketType.Pub, Bind("tcp://127.0.0.1:1234"))
```

上例将在本机的1234端口创建一个ZeroMQ发布者socket。.

类似的你可以创建一个带有监听器的订阅socket, 从发布者订阅所有的消息:

```scala
import akka.zeromq._
val listener = system.actorOf(Props(new Actor {
  def receive: Receive = {
    case Connecting    ⇒ //...
    case m: ZMQMessage ⇒ //...
    case _             ⇒ //...
  }
}))
val subSocket = system.newSocket(SocketType.Sub, Listener(listener), Connect("tcp://127.0.0.1:1234"), SubscribeAll)
```

下面的章节将描述所支持的连接方式以及它们在Akka环境中的用法。但是，如果要详细了解各种连接方式，请参阅 ZeroMQ – The Guide.

#####发布-订阅型连接
在一个发布-订阅(pub-sub)型连接中, 一个发布者可以有多个订阅者. 每个订阅者订阅一个或多个主题 , 发布者向主题发布消息. 订阅者也可以订阅所有的主题. 在 Akka 环境中, 如果要向并不直接与某actor打交道的actor发布消息，需要使用pub-sub 连接.

在使用 zeromq pub/sub 时你必须知道它需要多播 - 检查你的云环境 - 才能正确工作，同时对事件主题的过滤发生在客户端, 因此所有的事件都会广播到所有的订阅者.

actor订阅主题的过程如下:

```scala
val subTopicSocket = system.newSocket(SocketType.Sub, Listener(listener), Connect("tcp://127.0.0.1:1234"), Subscribe("foo.bar"))
```

订阅时使用前缀匹配，所以它订阅了所有以 foo.bar 开头的主题. 注意，如果没有提供主题名称或使用了 SubscribeAll , actor 将订阅所有的主题.

要取消订阅:

```scala
subTopicSocket ! Unsubscribe("foo.bar")
```

要向主题发布消息你必须使用两个 Frames，第一个Frame是主题.

```scala
pubSocket ! ZMQMessage(Seq(Frame("foo.bar"), Frame(payload)))
```

#####Pub-Sub实战
下例演示了一个有两个订阅者的发布者.

发布者监视当前的堆使用量和系统负载并周期性在"health.heap" 主题上发布 Heap 事件， 在"health.load"主题上发布 Load 事件.

```scala
import akka.zeromq._
import akka.actor.Actor
import akka.actor.Props
import akka.actor.ActorLogging
import akka.serialization.SerializationExtension
import java.lang.management.ManagementFactory

case object Tick
case class Heap(timestamp: Long, used: Long, max: Long)
case class Load(timestamp: Long, loadAverage: Double)

class HealthProbe extends Actor {

  val pubSocket = context.system.newSocket(SocketType.Pub, Bind("tcp://127.0.0.1:1235"))
  val memory = ManagementFactory.getMemoryMXBean
  val os = ManagementFactory.getOperatingSystemMXBean
  val ser = SerializationExtension(context.system)

  override def preStart() {
    context.system.scheduler.schedule(1 second, 1 second, self, Tick)
  }

  override def postRestart(reason: Throwable) {
    // 还要调用preStart，仅调度一次
  }

  def receive: Receive = {
    case Tick ⇒
      val currentHeap = memory.getHeapMemoryUsage
      val timestamp = System.currentTimeMillis

      // 用 akka SerializationExtension 转换成字节
      val heapPayload = ser.serialize(Heap(timestamp, currentHeap.getUsed, currentHeap.getMax)).fold(throw _, identity)
      // 第一Frame是话题，第二Frame是消息
      pubSocket ! ZMQMessage(Seq(Frame("health.heap"), Frame(heapPayload)))

      // 用 akka SerializationExtension 转换成字节
      val loadPayload = ser.serialize(Load(timestamp, os.getSystemLoadAverage)).fold(throw _, identity)
      // 第一Frame是主题，第二Frame是消息
      pubSocket ! ZMQMessage(Seq(Frame("health.load"), Frame(loadPayload)))
  }
}

  system.actorOf(Props[HealthProbe], name = "health")
```

我们添加一个订阅者来记录日志. 它订阅所有以 "health"开头的主题, i.e. 包括 Heap 和 Load 事件.

```scala
class Logger extends Actor with ActorLogging {

  context.system.newSocket(SocketType.Sub, Listener(self), Connect("tcp://127.0.0.1:1235"), Subscribe("health"))
  val ser = SerializationExtension(context.system)
  val timestampFormat = new SimpleDateFormat("HH:mm:ss.SSS")

  def receive = {
    // 第一Frame是主题，第二Frame是消息
    case m: ZMQMessage if m.firstFrameAsString == "health.heap" ⇒
      ser.deserialize(m.payload(1), classOf[Heap]) match {
        case Right(Heap(timestamp, used, max)) ⇒
          log.info("Used heap {} bytes, at {}", used, timestampFormat.format(new Date(timestamp)))
        case Left(e) ⇒ throw e
      }

    case m: ZMQMessage if m.firstFrameAsString == "health.load" ⇒
      ser.deserialize(m.payload(1), classOf[Load]) match {
        case Right(Load(timestamp, loadAverage)) ⇒
          log.info("Load average {}, at {}", loadAverage, timestampFormat.format(new Date(timestamp)))
        case Left(e) ⇒ throw e
      }
  }
}

  system.actorOf(Props[Logger], name = "logger")
```

另一个订阅者了解堆内存用量，如果堆使用过量则提出警告. 它仅订阅 Heap 事件.

```scala
class HeapAlerter extends Actor with ActorLogging {

  context.system.newSocket(SocketType.Sub, Listener(self), Connect("tcp://127.0.0.1:1235"), Subscribe("health.heap"))
  val ser = SerializationExtension(context.system)
  var count = 0

  def receive = {
    // 第一Frame是主题，第二Frame是消息
    case m: ZMQMessage if m.firstFrameAsString == "health.heap" ⇒
      ser.deserialize(m.payload(1), classOf[Heap]) match {
        case Right(Heap(timestamp, used, max)) ⇒
          if ((used.toDouble / max) > 0.9) count += 1
          else count = 0
          if (count > 10) log.warning("Need more memory, using {} %", (100.0 * used / max))
        case Left(e) ⇒ throw e
      }
  }
}

  system.actorOf(Props[HeapAlerter], name = "alerter")
```

#####Router-Dealer Connection
虽然 Pub/Sub 是很好的连接方式但zeromq的真正优势在于它象一个用于可靠消息通信的 “乐高玩具” . 而由于有如此多的集成方式，它的多语言支持是极好的. 当你使用ZeroMQ来集成多个系统时你可能需要创建自己的ZeroMQ机制。这时就轮到router 和 dealer socket 类型发挥作用了. 使用这些socket类型你可以创建自己的使用TCP/IP的可靠pub sub broker，并实现发布方的事件过滤.

要创建配置了高水位的Router socket:

```scala
val highWatermarkSocket = system.newSocket(
  SocketType.Router,
  Listener(listener),
  Bind("tcp://127.0.0.1:1234"),
  HighWatermark(50000))
```

akka-zeromq 模块支持大部分zeromq socket的配置选项.

#####推－拉型连接
Akka ZeroMQ 模块支持 推-拉 型连接.

创建 Push 连接:

```scala
def newPushSocket(socketParameters: Array[SocketOption]): ActorRef
```

创建 Pull 型连接:

```scala
def newPullSocket(socketParameters: Array[SocketOption]): ActorRef
```

很快将提供有更多的文档和示例.

#####请求－响应型连接
Akka ZeroMQ 模块支持 请求-响应 型连接.

创建 响应 连接:

```scala
def newReqSocket(socketParameters: Array[SocketOption]): ActorRef
```

创建 请求 连接:

```scala
def newRepSocket(socketParameters: Array[SocketOption]): ActorRef
```

很快将提供更多的文档和示例.
