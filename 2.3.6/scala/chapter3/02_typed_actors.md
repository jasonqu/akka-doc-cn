# 有类型Actor

有类型Actor是[Active Objects 模式](http://en.wikipedia.org/wiki/Active_object)的一种实现。Smalltalk诞生之时，就已经缺省地将方法调用从同步操作换为异步派发。

有类型Actor由两 “部分” 组成, 一个公开的接口和一个实现, 如果你有“企业级”Java的开发经验, 则应该非常熟悉。 对普通actor来说，你拥有一个外部API（公开接口的实例）来将方法调用异步地委托给其实现的私有实例。

有类型Actor相对于普通Actor的优势在于有类型Actor拥有静态的契约，你不需要定义你自己的消息；它的劣势在于对你能做什么和不能做什么进行了一些限制，即你不能使用``become/unbecome``。

有类型Actor是使用[JDK Proxies](http://docs.oracle.com/javase/6/docs/api/java/lang/reflect/Proxy.html)实现的，JDK Proxies提供了非常简单的api来拦截方法调用。

> 注意

> 和普通Akka actor一样，有类型actor一次也只处理一个消息。

###何时使用有类型actor
有类型actor是桥接actor系统（"内部"）和非actor代码 （"外部"）的良好方式，因为它们允许你在外部编写普通OO式代码。把它们看做大门：其实用性在于私有领域和公共接口之间，而你不想你的房子内部有太多的门，不是吗？更长的讨论请参见[这篇博客](http://letitcrash.com/post/19074284309/when-to-use-typedactors)。

更多的背景：TypedActors可以很容易被滥用作RPC，它们都是一个抽象概念，[众所周知](http://doc.akka.io/docs/misc/smli_tr-94-29.pdf)是有缺陷的。因此当我们容易和正确的编写高度可扩展的并行软件时，TypedActors并非首选。他们有自己的定位，必要时才使用它们。

###工具箱
在创建第一个有类型Actor之前，我们先了解一下我们手上可供使用的工具，它位于``akka.actor.TypedActor``中。

```scala
import akka.actor.TypedActor
 
//返回有类型actor扩展
val extension = TypedActor(system) //system是一个Actor系统实例
 
//判断一个引用是否是有类型actor代理
TypedActor(system).isTypedActor(someReference)
 
//返回一个外部有类型actor代理所代表的Akka actor
TypedActor(system).getActorRefFor(someReference)
 
//返回当前的ActorContext,
// 此方法仅在一个TypedActor 实现的方法中有效
val c: ActorContext = TypedActor.context
 
//返回当前有类型actor的外部代理,
// 此方法仅在一个TypedActor 实现的方法中有效
val s: Squarer = TypedActor.self[Squarer]
 
//返回一个有类型Actor扩展的上下文实例
//这意味着如果你用它创建其它的有类型actor，它们会成为当前有类型actor的子actor
TypedActor(TypedActor.context)
```

> 警告

> 就象不应该暴露Akka actor的``this``一样，不要暴露有类型Actor的``this``，你应该传递其外部代理引用，它可以在你的有类型Actor中用``TypedActor.self``获得, 这是你的外部标识, 就象``ActorRef``是Akka actor的外部标识一样。

###创建有类型Actor
要创建有类型Actor，需要一个或多个接口，和一个实现。

我们的示例接口:

```scala
trait Squarer {
  def squareDontCare(i: Int): Unit //fire-forget
 
  def square(i: Int): Future[Int] //non-blocking send-request-reply
 
  def squareNowPlease(i: Int): Option[Int] //blocking send-request-reply
 
  def squareNow(i: Int): Int //blocking send-request-reply
 
  @throws(classOf[Exception]) //declare it or you will get an UndeclaredThrowableException
  def squareTry(i: Int): Int //blocking send-request-reply with possible exception
}
```

好，现在我们有了一些可以调用的方法，但我们需要在SquarerImpl中实现。

```scala
class SquarerImpl(val name: String) extends Squarer {
 
  def this() = this("default")
  def squareDontCare(i: Int): Unit = i * i //Nobody cares :(
 
  def square(i: Int): Future[Int] = Future.successful(i * i)
 
  def squareNowPlease(i: Int): Option[Int] = Some(i * i)
 
  def squareNow(i: Int): Int = i * i
 
  def squareTry(i: Int): Int = throw new Exception("Catch me!")
}
```

太好了，我们现在有了接口，也有了对这个接口的实现，我们还知道如何从他们来创建一个有类型actor，现在我们来看看如何调用这些方法。

创建我们的Squarer的有类型actor实例的最简单方法是：

```scala
val mySquarer: Squarer =
  TypedActor(system).typedActorOf(TypedProps[SquarerImpl]())
```

第一个类型是代理的类型，第二个类型是实现的类型。如果要调用某特定的构造方法要这样做：

```scala
val otherSquarer: Squarer =
  TypedActor(system).typedActorOf(TypedProps(classOf[Squarer],
    new SquarerImpl("foo")), "name")
```

由于你提供了一个 Props, 你可以指定使用哪个派发器, 缺省的超时时间等。

###方法派发语义
方法返回:

* ``Unit`` 会以 ``fire-and-forget``语义进行派发，与``ActorRef.tell``完全一致。
* ``akka.dispatch.Future[_]`` 会以 ``send-request-reply``语义进行派发，与 ``ActorRef.ask``完全一致。
* ``scala.Option[_]``会以``send-request-reply``语义派发，但是*会*阻塞等待应答, 如果在超时时限内没有应答则返回``scala.None``，否则返回包含结果的``scala.Some[_]``。在这个调用中发生的异常将被重新抛出。
* 任何其它类型的值将以``send-request-reply``语义进行派发，但*会*阻塞地等待应答, 如果超时会抛出``java.util.concurrent.TimeoutException``，如果发生异常则将异常重新抛出。

###消息与不可变性
虽然Akka不能强制要求你传给有类型Actor方法的参数类型是不可变的, 我们*强烈*建议只传递不可变参数。

#####单向消息发送

```scala
mySquarer.squareDontCare(10)
```

就是这么简单！方法会在另一个线程中异步地调用。

#####请求-响应消息发送

```scala
val oSquare = mySquarer.squareNowPlease(10) //Option[Int]
```

如果需要，这会阻塞到有类型actor的Props中设置的超时时限。如果超时，会返回``None`` 。

```scala
val iSquare = mySquarer.squareNow(10) //Int
```

如果需要，这会阻塞到有类型actor的Props中设置的超时时限。如果超时，会抛出``java.util.concurrent.TimeoutException``。

#####请求-以future作为响应的消息发送

```scala
val fSquare = mySquarer.square(10) //A Future[Int]
```

这个调用是异步的，返回的Future可以用作异步组合。

###终止有类型Actor
由于有类型actor底层还是Akka actor，所以在不需要的时候要终止它。

```scala
TypedActor(system).stop(mySquarer)
```

这将会尽快地异步终止与指定的代理关联的有类型Actor。

```scala
TypedActor(system).poisonPill(otherSquarer)
```

这将会在有类型actor完成所有入队的调用后异步地终止它。

###有类型Actor监管树
你可以通过传入一个``ActorContext``来获得有类型Actor上下文，所以你可以对它调用``typedActorOf(..)``来创建有类型子actor。

```scala
//Inside your Typed Actor
val childSquarer: Squarer =
  TypedActor(TypedActor.context).typedActorOf(TypedProps[SquarerImpl]())
//Use "childSquarer" as a Squarer
```

通过将``ActorContext``作为参数传给``TypedActor.get(…)``，也可以为普通的Akka actor创建有类型子actor。

###监管策略
通过让你的有类型Actor的具体实现类实现``TypedActor.Supervisor``方法，你可以定义用来监管子actor的策略，就像[监管与监控](../chapter2/04_supervision_and_monitoring.md) 和[容错(Scala)](03_fault_tolerance.md)所描述的。

###生命周期回调
通过使你的有类型actor实现类实现以下方法:

* ``TypedActor.PreStart``
* ``TypedActor.PostStop``
* ``TypedActor.PreRestart``
* ``TypedActor.PostRestart``

你可以hook进有类型actor的整个生命周期。

###接收任意消息
如果你的有类型actor的实现类扩展了``akka.actor.TypedActor.Receiver``，所有非方法调用``MethodCall``的消息会被传给``onReceive``方法.

这使你能够对DeathWatch的``Terminated``消息及其它类型的消息进行处理，例如，与无类型actor进行交互的场合。

###代理
你可以使用带TypedProps和ActorRef参数的``typedActorOf``来将指定的Actor引用代理成一个有类型Actor。这在你需要与远程主机上的有类型Actor通信时会有用, 只要将``ActorRef``传递给 ``typedActorOf``即可。

> 注意

> 目标Actor引用需要能处理``MethodCall``消息.

###查找与远程处理
因为``TypedActor``底层还是``Akka Actors``，你可以使用``typedActorOf``来代理可能在远程节点上的``ActorRefs``。

```scala
val typedActor: Foo with Bar =
  TypedActor(system).
    typedActorOf(
      TypedProps[FooBar],
      actorRefToRemoteActor)
//Use "typedActor" as a FooBar
```

###功能扩充
以下是使用traits来为你的有类型actor混入行为的示例：

```scala
trait Foo {
  def doFoo(times: Int): Unit = println("doFoo(" + times + ")")
}
 
trait Bar {
  def doBar(str: String): Future[String] =
    Future.successful(str.toUpperCase)
}

class FooBar extends Foo with Bar
```

```scala
val awesomeFooBar: Foo with Bar =
  TypedActor(system).typedActorOf(TypedProps[FooBar]())
 
awesomeFooBar.doFoo(10)
val f = awesomeFooBar.doBar("yes")
 
TypedActor(system).poisonPill(awesomeFooBar)
```

###有类型路由器模式
有时你想要传播多个actor之间的消息。在Akka中实现这一目标的最简单方法是使用一个[路由器](06_routing.md)，可以实现特定的路由逻辑，例如最小邮箱``smallest-mailbox``或一致性哈希``consistent-hashing``等。

路由器不能直接提供给有类型actor，但可以很容易的利用非类型化的路由器，并在其使用一个有类型代理即可。为了展示，让我们创建有类型actor并分配它们一些随机``id``，所以我们知道事实上，路由器已向消息发送给不同的actor：

```scala
trait HasName {
  def name(): String
}
 
class Named extends HasName {
  import scala.util.Random
  private val id = Random.nextInt(1024)
 
  def name(): String = "name-" + id
}
```

为了在此类actor的几个实例中轮询访问（round robin），你可以简单地创建一个普通的非类型化路由器，然后像下面的示例所示把它包装为一个``TypedActor``。这之所以能够正确工作，是因为有类型actor与普通actor使用相同的机制通讯，其方法调用最终都被转换为``MethodCall``消息的发送。

```scala
def namedActor(): HasName = TypedActor(system).typedActorOf(TypedProps[Named]())
 
// prepare routees
val routees: List[HasName] = List.fill(5) { namedActor() }
val routeePaths = routees map { r =>
  TypedActor(system).getActorRefFor(r).path.toStringWithoutAddress
}
 
// prepare untyped router
val router: ActorRef = system.actorOf(RoundRobinGroup(routeePaths).props())
 
// prepare typed proxy, forwarding MethodCall messages to `router`
val typedRouter: HasName =
  TypedActor(system).typedActorOf(TypedProps[Named](), actorRef = router)
 
println("actor was: " + typedRouter.name()) // name-184
println("actor was: " + typedRouter.name()) // name-753
println("actor was: " + typedRouter.name()) // name-320
println("actor was: " + typedRouter.name()) // name-164
```

