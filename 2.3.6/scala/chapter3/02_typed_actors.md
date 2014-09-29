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
* ``scala.Option[_] `` 会以 ``send-request-reply`` 语义派发，但是*会*阻塞等待应答, 如果在超时时限内没有应答则返回``scala.None``，否则返回包含结果的``scala.Some[_]``。在这个调用中发生的异常将被重新抛出。
* 任何其它类型的值将以``send-request-reply``语义进行派发，但*会*阻塞地等待应答, 如果超时会抛出``java.util.concurrent.TimeoutException``，如果发生异常则将异常重新抛出。



