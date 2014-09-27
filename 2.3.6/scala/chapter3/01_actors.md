# Actors

[Actor模型](http://en.wikipedia.org/wiki/Actor_model)为编写并发和分布式系统提供了一种更高的抽象级别。它将开发人员从显式地处理锁和线程管理的工作中解脱出来，使编写并发和并行系统更加容易。Actor模型是在1973年Carl Hewitt的论文中定义的，不过直到被Erlang语言采用后才变得流行起来，一个成功案例是爱立信使用Erlang非常成功地创建了高并发的可靠的电信系统。

Akka Actor的API与Scala Actor类似，它们都借鉴了Erlang的一些语法。

###创建Actor

> 注意

> 由于Akka采用强制性的父子监管，每一个actor都被监管着，并且（可能是）别的actor的监管者；我们建议你熟悉一下[Actor系统](../chapter2/02_actor_systems.md) 和 [监管与监控](../chapter2/04_supervision_and_monitoring.md)，阅读 [Actor引用, 路径与地址](../chapter2/05_actor_references_paths_and_addresses.md)也有帮助。

#####定义一个Actor类
要定义自己的Actor类，需要继承`Actor`并实现`receive`方法。`receive`方法需要定义一系列case语句（类型为``PartialFunction[Any, Unit]``）来描述你的Actor能够处理哪些消息（使用标准的Scala模式匹配），以及消息如何被处理。

如下例：

```scala
import akka.actor.Actor
import akka.actor.Props
import akka.event.Logging
 
class MyActor extends Actor {
  val log = Logging(context.system, this)
  def receive = {
    case "test" => log.info("received test")
    case _      => log.info("received unknown message")
  }
}
```

请注意Akka Actor的`receive`消息循环是完全的，这与Erlang和Scala的Actor行为不同。这意味着你需要提供一个模式匹配，来处理这个actor所能够接受的所有消息的规则，如果你希望处理未知的消息，你需要象上例一样提供一个缺省的case分支。否则会有一个``akka.actor.UnhandledMessage(message,
sender, recipient)``被发布到Actor系统（`ActorSystem`）的事件流（`EventStream`）中。

进一步注意到上面定义行为的返回类型是``Unit``；如果该actor应该回应收到的消息，则必须明确写出，如下文所述。

`receive`方法的结果是一个偏函数对象，它存储在actor中作为其"最初的行为"，对actor构造完成后改变行为的进一步信息，请参阅[Become/Unbecome](#become-unbecome)。

#####<a name="props"></a>Props
`Props`是一个用来在创建actor时指定选项的配置类，可以把它看作是不可变的，因此在创建包含相关部署信息的actor时（例如使用哪一个调度器[dispatcher]，详见下文），是可以自由共享的。以下是使用如何创建`Props`实例的示例.

```scala
import akka.actor.Props
 
val props1 = Props[MyActor]
val props2 = Props(new ActorWithArgs("arg")) // careful, see below
val props3 = Props(classOf[ActorWithArgs], "arg")
```

第二个变量`props2`演示了创建actor时，怎样将构造函数参数传进去，但它只应在actor之外使用，详见下文。

最后一行展示了一种可能性，它在不关注上下文的情况下传递构造函数参数。在构造`Props`对象的时候，会检查是否存在匹配的构造函数，如果没有或存在多个匹配的构造函数，则会导致`IllegalArgumentEception`。

#####危险的变量

```scala
// 不建议在另一个actor内使用:
// 因为它鼓励封闭作用域
val props7 = Props(new MyActor)
```

该方法不建议在另一个actor内使用，因为它鼓励封闭作用域，从而导致不可序列化的`Props`和可能的竞态条件（破坏了actor封装）。我们将提供一个基于宏的解决方案，在将来的版本中没有头痛地支持类似的语法，届时该变量将被恰当的废弃。另一方面在actor的伴生对象下，在一个`Props`工厂中使用该变量是完全没问题的，如下面"推荐做法"所述。

这些方法有两个用例：将构造函数的参数传递给该actor——由新推出的`Props.apply(clazz, args)`方法解决了，如上面或下面的推荐做法所示——和作为匿名类创建"临时性"的actor。后者应通过命名这些actor，而非匿名来解决（如果他们未在顶级``object``中声明，则封闭的实例的``this``引用需要作为第一个参数传入）。

> 警告

> 在另一个actor中声明一个actor是非常危险的，会打破actor的封装。永远不要将一个actor的``this``引用传进`Props`！

#####推荐做法
在每一个`Actor`的伴生对象中提供工厂方法是一个好主意，这有助于保持创建合适的`Props`，尽可能接近actor的定义。这也避免了使用``Props.apply(...)``方法将采用一个“按名”（by-name）参数的缺陷，因为伴生对象的给定代码块中将不会保留包含作用域的引用：

```scala
object DemoActor {
  /**
   * Create Props for an actor of this type.
   * @param magciNumber The magic number to be passed to this actor’s constructor.
   * @return a Props for creating this actor, which can then be further configured
   *         (e.g. calling `.withDispatcher()` on it)
   */
  def props(magicNumber: Int): Props = Props(new DemoActor(magicNumber))
}
 
class DemoActor(magicNumber: Int) extends Actor {
  def receive = {
    case x: Int => sender() ! (x + magicNumber)
  }
}
 
class SomeOtherActor extends Actor {
  // Props(new DemoActor(42)) would not be safe
  context.actorOf(DemoActor.props(42), "demo")
  // ...
}
```

#####使用Props创建Actor
Actor可以通过将`Props`实例传入`actorOf`工厂方法来创建，`ActorSystem`和`ActorContext`中都有该方法。

```scala
import akka.actor.ActorSystem
 
// ActorSystem is a heavy object: create only one per application
val system = ActorSystem("mySystem")
val myActor = system.actorOf(Props[MyActor], "myactor2")
```

使用`ActorSystem`将创建顶级actor，由actor系统提供监督actor监管，如果使用一个actor的上下文，则创建其一个子actor。

```scala
class FirstActor extends Actor {
  val child = context.actorOf(Props[MyActor], name = "myChild")
  // plus some behavior ...
}
```

推荐创建一个树形结构，包含子actor、孙子等等，使之符合应用的逻辑错误处理结构，见[Actor系统](../chapter2/02_actor_systems.md)。

对`actorOf`的调用返回一个`ActorRef`实例。它是actor实例的句柄，并且是与之进行交互的唯一方法。`ActorRef`是不可变的，与其代表的actor有一一对应关系。`ActorRef`也是可序列化，并能通过网络传输。这意味着你可以将其序列化，通过网线发送，并在远程主机上使用它，而且虽然跨网络，它将仍然代表在原始节点上相同的actor。

名称参数是可选的，不过你应该良好命名你的actor，因为名称将被用于日志打印和标识actor。名称不能为空，且不能以``$``开头，不过它可以包含URL编码字符（如空格用``%20``表示）。如果给定的名称已经被同一个父亲下的另一个子actor使用，则会跑出一个`InvalidActorNameException`。

actor在创建时，会自动异步启动。

#####依赖注入
如果你的actor都有一个带参数的构造函数，它们也需要成为`Props`的一部分，如[上文](#props)所述。但有些情况下必须使用工厂方法，例如，当实际构造函数的参数由依赖注入框架决定。

```scala
import akka.actor.IndirectActorProducer
 
class DependencyInjector(applicationContext: AnyRef, beanName: String)
  extends IndirectActorProducer {
 
  override def actorClass = classOf[Actor]
  override def produce =
    // obtain fresh Actor instance from DI framework ...
}
 
val actorRef = system.actorOf(
  Props(classOf[DependencyInjector], applicationContext, "hello"),
  "helloBean")
```

> 警告

> 你有时可能倾向于提供一个`IndirectActorProducer`始终返回相同的实例，例如通过使用``lazy val``。这是不受支持的，因为它违背了[actor重启的意义](../chapter2/04_supervision_and_monitoring.md#supervision-restart)。

> 当使用依赖注入框架时，actor bean*必须不是*单例作用域。

关于依赖注入极其Akka集成的更多内容情参见[“在Akka中使用依赖注入”](http://letitcrash.com/post/55958814293/akka-dependency-injection)指南和Typesafe Activator的[“Akka Java Spring”](http://www.typesafe.com/activator/template/akka-java-spring)教程。

#####收件箱
当在actor外编写与actor交互的代码时，``ask``模式可以作为一个解决方案（见下文），但有两件事它不能做：接收多个答复（例如将`ActorRef`订阅到一个通知服务）和查看其他actor的生命周期。为达到这些目的可以使用`Inbox`类：

```scala
implicit val i = inbox()
echo ! "hello"
i.receive() should be("hello")
```

有一个从收件箱到actor引用的一个隐式转换，意味着在这个例子中，发送者引用将成为收件箱隐藏的actor。这就允许了收件箱收到回应，如最后一行所示。监控一个actor也相当简单：

```scala
val target = // some actor
val i = inbox()
i watch target
```

###Actor API
`Actor` trait只定义了一个抽象方法，就是上面提到的`receive`，用来实现actor的行为。

如果当前actor的行为与收到的消息不匹配，则会调用 `unhandled`，其缺省实现是向actor系统的事件流中发布一条``akka.actor.UnhandledMessage(message, sender, recipient)``（将配置项目`akka.actor.debug.unhandled`设置为``on``来将它们转换为实际的调试消息）。

另外，它还包括:

* `self`引用代表本actor的`ActorRef`
* `sender`引用代表最近收到消息的发送actor，通常用于下面将讲到的[消息回应](#actor-reply)中
* `supervisorStrategy` 用户可重写它来定义对子actor的监管策略

  该策略通常在actor内声明，这样决定函数就可以访问actor的内部状态：因为失败通知作为消息发送给监管者，并像普通消息一样被处理（尽管不是正常行为），所有的值和actor变量都是可用的，以及``sender``引用 （这将报告失败的直接子actor；如果原始失败发生在遥远的后裔，它仍然是一次向上报告一层）。

* `context`暴露actor和当前消息的上下文信息，如：
  * 用于创建子actor的工厂方法（`actorOf`）
  * actor所属的系统
  * 父监管者
  * 所监管的子actor
  * 生命周期监控
  * hotswap行为栈，见[Become/Unbecome](#actor-hotswap)

你可以import `context`的成员来避免总加上``context.``前缀

```scala
class FirstActor extends Actor {
  import context._
  val myActor = actorOf(Props[MyActor], name = "myactor")
  def receive = {
    case x => myActor ! x
  }
}
```

其余的可见方法是可以被用户重写的生命周期hook，描述如下:

```scala
def preStart(): Unit = ()
 
def postStop(): Unit = ()
 
def preRestart(reason: Throwable, message: Option[Any]): Unit = {
  context.children foreach { child ⇒
    context.unwatch(child)
    context.stop(child)
  }
  postStop()
}
 
def postRestart(reason: Throwable): Unit = {
  preStart()
}
```

以上所示的实现是`Actor` trait的缺省实现。

###Actor生命周期

.. image:: ../images/actor_lifecycle.png
   :align: center
   :width: 680

actor系统中的路径代表一个"地方"，这里可能会被活着的actor占据。最初（除了系统初始化actor）路径都是空的。在调用``actorOf()``时它将为指定路径分配actor根据传入``Props``创建的一个*化身*。actor化身是由路径*和一个UID*标识的。重新启动只会替换有``Props``定义的``Actor``实例，但不会替换化身，因此UID保持不变。

当actor停止时，其化身的生命周期结束。在这一点上相关的生命周期事件被调用，观察中的actor都会获得终止通知。当化身停止后，路径可以重复使用，通过`actorOf()`创建一个actor。在这种情况下，除了UID不同外，新化身与老化身是相同的。

`ActorRef`始终表示的化身（路径和UID）而不只是一个给定的路径。因此如果actor停止，并且创建一个新的具有相同名称的actor，则新指向老化身的`ActorRef`将不会指向新的化身。

相对地，`ActorSelection`指向路径（或多个路径，如果使用了通配符），且完全不关注有没有化身占据它。因此`ActorSelection` 不能被观察。获取某路径下的当前化身``ActorRef``是可能的，只要向该``ActorSelection``发送``Identify``，如果收到``ActorIdentity``回应，则正确的引用就包含其中（详见[通过Actor Selection确定Actor](#actorselection-scala)）。也可以使用`ActorSelection`的``resolveOne``方法，它会返回一个包含匹配`ActorRef`的``Future``。

#####使用DeathWatch进行生命周期监控
为了在其它actor终止时 (即永久停止, 而不是临时的失败和重启)收到通知, actor可以将自己注册为其它actor在终止时所发布的`Terminated`消息的接收者（见[停止 Actor](#stopping-actors)）。这个服务是由actor系统的`DeathWatch`组件提供的。

注册一个监控器很简单：

```scala
import akka.actor.{ Actor, Props, Terminated }
 
class WatchActor extends Actor {
  val child = context.actorOf(Props.empty, "child")
  context.watch(child) // <-- this is the only call needed for registration
  var lastSender = system.deadLetters
 
  def receive = {
    case "kill" =>
      context.stop(child); lastSender = sender()
    case Terminated(`child`) => lastSender ! "finished"
  }
}
```

要注意`Terminated`消息的产生与注册和终止行为所发生的顺序无关。特别地，即使在注册时，被观察的actor已经终止了，观察actor仍然会受到一个`Terminated`消息。

多次注册并不表示会有多个消息产生，也不保证有且只有一个这样的消息被接收到：如果被监控的actor已经生成了消息并且已经进入了队列，在这个消息被处理之前又发生了另一次注册，则会有第二个消息进入队列，因为对一个已经终止的actor的注册会监控操作会立刻导致`Terminated`消息的产生。

可以使用``context.unwatch(target)``来停止对另一个actor生存状态的监控。即使`Terminated`已经加入邮箱，该操作仍有效；一旦调用`unwatch`，则被观察的actor的`Terminated`消息都不会再被处理。

#####启动Hook
actor启动后，它的`preStart`方法会被立即执行。

```scala
override def preStart() {
  // registering with other actors
  someService ! Register(self)
}
```

这位actor第一次创建时，将调用此方法。在重新启动期间，它被`postRestart`的默认实现调用，这意味着通过重写该方法，你可以选择是仅仅在初始化该actor时调用一次，还是为每次重新启动都调用。actor构造函数中的初始化代码将在每个actor实例创建的时候被调用，这也发生在每次重启时。

#####重启 Hook
所有的actor都是被监管的，即与另一个使用某种失败处理策略的actor链接在一起。如果在处理一个消息的时候抛出的异常，Actor将被重启（详见[监管与监控](../chapter2/04_supervision_and_monitoring.md)）。这个重启过程包括上面提到的Hook:



TODO
1. 要被重启的actor的 preRestart 被调用，携带着导致重启的异常以及触发异常的消息; 如果重启并不是因为消息的处理而发生的，所携带的消息为 None , 例如，当一个监管者没有处理某个异常继而被它自己的监管者重启时。 这个方法是用来完成清理、准备移交给新的actor实例的最佳位置。 它的缺省实现是终止所有的子actor并调用 postStop.
最初 actorOf 调用的工厂方法将被用来创建新的实例。
新的actor的 postRestart 方法被调用，携带着导致重启的异常信息。 By default the preStart is called, just as in the normal start-up case.
actor的重启会替换掉原来的actor对象; 重启不影响邮箱的内容, 所以对消息的处理将在 postRestart hook 返回后继续. 触发异常的消息不会被重新接收。在actor重启过程中所有发送到该actor的消息将象平常一样被放进邮箱队列中。


























































