# Akka扩展

> 注：本节未经校验，如有问题欢迎提issue

如果想要为Akka添加特性，有一个非常优美而且强大的工具，称为 Akka 扩展。它由两部分组成： ``Extension`` 和 ``ExtensionId``.

Extensions 在每个 ``ActorSystem`` 中只会加载一次, 并被Akka所管理。 你可以选择按需加载你的Extension或是在 ``ActorSystem`` 创建时通过Akka配置来加载。 关于这些细节，见下文 “从配置中加载” 的部分.

> 警告

> 由于扩展是hook到Akka自身的，所以扩展的实现者需要保证自己扩展的线程安全性。

###构建一个扩展
现在我们来创建一个扩展示例，它的功能是对某件事发生的次数进行统计。

首先定义 ``Extension`` 的功能:

```scala
import akka.actor.Extension

class CountExtensionImpl extends Extension {
  //Since this Extension is a shared instance
  // per ActorSystem we need to be threadsafe
  private val counter = new AtomicLong(0)

  //This is the operation this Extension provides
  def increment() = counter.incrementAndGet()
}
```

然后需要为扩展指定一个 ``ExtensionId``，这样我们可以获取它的实例.

```scala
import akka.actor.ActorSystem
import akka.actor.ExtensionId
import akka.actor.ExtensionIdProvider
import akka.actor.ExtendedActorSystem

object CountExtension
  extends ExtensionId[CountExtensionImpl]
  with ExtensionIdProvider {
  //The lookup method is required by ExtensionIdProvider,
  // so we return ourselves here, this allows us
  // to configure our extension to be loaded when
  // the ActorSystem starts up
  override def lookup = CountExtension

  //This method will be called by Akka
  // to instantiate our Extension
  override def createExtension(system: ExtendedActorSystem) = new CountExtensionImpl

  /**
   * Java API: retrieve the Count extension for the given system.
   */
  override def get(system: ActorSystem): CountExtensionImpl = super.get(system)
}
```

好了！然后我们就可以使用它了:

```scala
CountExtension(system).increment
```

或者在Akka Actor中使用:

```scala
class MyActor extends Actor {
  def receive = {
    case someMessage =>
      CountExtension(context.system).increment()
  }
}
```

你也可以将扩展藏在 trait 里:

```scala
trait Counting { self: Actor =>
  def increment() = CountExtension(context.system).increment()
}
class MyCounterActor extends Actor with Counting {
  def receive = {
    case someMessage => increment()
  }
}
```

这样就搞定了!

###从配置中加载
为了能够从Akka配置中加载扩展，你必须在为`ActorSystem`提供的配置文件中的 ``akka.extensions`` 部分加上 ``ExtensionId`` 或 ``ExtensionIdProvider``实现类的完整路径。

```
akka {
  extensions = ["docs.extension.CountExtension"]
}
```

###实用性
充分发挥你的想象力，天空才是极限! 顺便提一下，你知道 Akka的``Typed Actor``, ``Serialization``和其它一些特性都是以Akka扩展的形式实现的吗？

#####应用特定设置
可以用 [Configuration]() 来指定应用特有的设置。将这些设置放在一个扩展里是一个好习惯。

配置示例:

```
myapp {
  db {
    uri = "mongodb://example1.com:27017,example2.com:27017"
  }
  circuit-breaker {
    timeout = 30 seconds
  }
}
```

``Extension``的实现:

```scala
import akka.actor.ActorSystem
import akka.actor.Extension
import akka.actor.ExtensionId
import akka.actor.ExtensionIdProvider
import akka.actor.ExtendedActorSystem
import scala.concurrent.duration.Duration
import com.typesafe.config.Config
import java.util.concurrent.TimeUnit

class SettingsImpl(config: Config) extends Extension {
  val DbUri: String = config.getString("myapp.db.uri")
  val CircuitBreakerTimeout: Duration =
    Duration(config.getMilliseconds("myapp.circuit-breaker.timeout"),
      TimeUnit.MILLISECONDS)
}
object Settings extends ExtensionId[SettingsImpl] with ExtensionIdProvider {

  override def lookup = Settings

  override def createExtension(system: ExtendedActorSystem) =
    new SettingsImpl(system.settings.config)

  /**
   * Java API: retrieve the Settings extension for the given system.
   */
  override def get(system: ActorSystem): SettingsImpl = super.get(system)
}
```

使用它：

```scala
class MyActor extends Actor {
  val settings = Settings(context.system)
  val connection = connect(settings.DbUri, settings.CircuitBreakerTimeout)
```
