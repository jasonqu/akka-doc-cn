# 配置

你开始使用Akka时可以不需要定义任何配置，因为Akka提供了合理的默认值。不过为了适应特定的运行环境，你可能需要修改设置来更改默认行为。可以修改的典型设置的例子：

* 日志级别和后端记录器
* 启用远程
* 消息序列化
* 路由器定义
* 调度的调整

Akka使用[Typesafe配置库](https://github.com/typesafehub/config)，它同样也是你自己的应用或库的不错选择，不管你用不用Akka。这个库由Java实现，没有外部的依赖；本文后面将只对这个库有一些归纳，你应该查看其文档参考具体的使用（尤其是[ConfigFactory](http://typesafehub.github.io/config/v1.2.0/com/typesafe/config/ConfigFactory.html)）。


> 警告

> 如果你在Scala REPL的2.9.x系列版本中使用Akka，并且你不提供自己的ClassLoader给ActorSystem，则需要以“-Yrepl-sync”选项启动REPL来解决上下文ClassLoader的缺陷。

###配置读取的地方

Akka的所有配置都保存在`ActorSystem`的实例中，或者换一种说法，从外界来看，`ActorSystem`是配置信息的唯一消费者。在构造一个actor系统时，你可以选择传进一个`Config`对象，如果不传则等效于传入``ConfigFactory.load()``（通过正确的类加载器）。粗略的讲，这意味着默认会解析classpath根目录下所有的`application.conf`，`application.json`和`application.properties`文件——请参考前面提到的文档以获取细节。然后actor系统会合并classpath根目录下的所有 ``reference.conf``行成后备配置，也就是说，它在内部使用

```scala
  appConfig.withFallback(ConfigFactory.defaultReference(classLoader))
```

其哲学是代码永远不包含缺省值，相反是依赖于随库提供的 `reference.conf` 中的配置。

系统属性中覆盖的配置具有最高优先级，参见[HOCON 规范](https://github.com/typesafehub/config/blob/master/HOCON.md)（靠近末尾的位置）。此外值得注意的是，应用程序配置——缺省为``application``——可以使用``config.resource``属性重写 （还有更多，请参阅[配置文档](https://github.com/typesafehub/config/blob/master/README.md)）。

> 注意

> 如果你正在编写一个Akka 应用，将你的配置保存类路径的根目录下的``application.conf``文件中。如果你正在编写一个基于Akka的库，将其配置保存在JAR包根目录下的``reference.conf``文件中。

###当使用JarJar,，OneJar，Assembly或任何jar打包命令（jar-bundler）

> 警告

> Akka的配置方法重度依赖于这个理念——每一模块/jar都有它自己的 `reference.conf`文件，所有这些都将会被配置发现并加载。不幸的是，这也意味着如果你放置/合并多个jar到相同的 jar中，你也需要合并所有的`reference.conf`文件。否则所有的默认设置将会丢失，Akka将无法工作。

如果你使用 Maven 打包应用程序，你还可以使用[Apache Maven Shade Plugin](
http://maven.apache.org/plugins/maven-shade-plugin)中对[资源转换（Resource
Transformers）](http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#AppendingTransformer)的支持，来将所有构建类路径中的`reference.conf`合并为一个文件。

插件配置可能如下所示：

```xml
    <plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-shade-plugin</artifactId>
     <version>1.5</version>
     <executions>
      <execution>
       <phase>package</phase>
       <goals>
        <goal>shade</goal>
       </goals>
       <configuration>
        <shadedArtifactAttached>true</shadedArtifactAttached>
        <shadedClassifierName>allinone</shadedClassifierName>
        <artifactSet>
         <includes>
          <include>*:*</include>
         </includes>
        </artifactSet>
        <transformers>
          <transformer
           implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
           <resource>reference.conf</resource>
          </transformer>
          <transformer
           implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
           <manifestEntries>
            <Main-Class>akka.Main</Main-Class>
           </manifestEntries>
          </transformer>
        </transformers>
       </configuration>
      </execution>
     </executions>
    </plugin>
```

<span id="dakka-log-config-on-start"></span>
###自定义application.conf
一个自定义的``application.conf`可能看起来像这样：

```
  # In this file you can override any option defined in the reference files.
  # Copy in parts of the reference files and modify as you please.

  akka {

    # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
    # to STDOUT)
    loggers = ["akka.event.slf4j.Slf4jLogger"]

    # Log level used by the configured loggers (see "loggers") as soon
    # as they have been started; before that, see "stdout-loglevel"
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    loglevel = "DEBUG"

    # Log level for the very basic logger activated during ActorSystem startup.
    # This logger prints the log messages to stdout (System.out).
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    stdout-loglevel = "DEBUG"

    actor {
      provider = "akka.cluster.ClusterActorRefProvider"

      default-dispatcher {
        # Throughput for default Dispatcher, set to 1 for as fair as possible
        throughput = 10
      }
    }

    remote {
      # The port clients should connect to. Default is 2552.
      netty.tcp.port = 4711
    }
  }
```

###<a name="including-files"></a>包含文件
有时包含另一个配置文件内容的能力是非常有用的，例如假设你有一个``application.conf``包含所有环境独立设置，然后使用特定环境的设置覆写。

用``-Dconfig.resource=/dev.conf``制定系统属性，将会加载``dev.conf``文件，并包含``application.conf``

dev.conf:

```
  include "application"

  akka {
    loglevel = "DEBUG"
  }
```

更高级的包括和替换机制的解释在[HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md)规范中。


###配置日志
如果系统或配置属性``akka.log-config-on-start`` 被设置为 ``on``，则在actor系统启动的时候，就完成了INFO级别的日志设置。当你不能确定使用何种配置时，这很有用。

如果有疑问，你也可以在创建actor系统之前或之后，很容易很方便地检查配置对象：

```
  Welcome to Scala version @scalaVersion@ (Java HotSpot(TM) 64-Bit Server VM, Java 1.6.0_27).
  Type in expressions to have them evaluated.
  Type :help for more information.

  scala> import com.typesafe.config._
  import com.typesafe.config._

  scala> ConfigFactory.parseString("a.b=12")
  res0: com.typesafe.config.Config = Config(SimpleConfigObject({"a" : {"b" : 12}}))

  scala> res0.root.render
  res1: java.lang.String =
  {
      # String: 1
      "a" : {
          # String: 1
          "b" : 12
      }
  }
```

展示结果中，每个项目前会有评论展示这个配置的起源（对应的文件和行数），并展示已存在的评论，如配置参考中的。actor系统合并参考并解析后形成的设置，可以这样显示：

```java
  final ActorSystem system = ActorSystem.create();
  System.out.println(system.settings());
  // this is a shortcut for system.settings().config().root().render()
```

###谈一谈类加载器

在配置文件的几个地方，可以通过制定类的全名来让Akka实例化该类。这是通过Java反射完成的，相应地用到了一个`ClassLoader`。在具有挑战性的环境中，如应用容器和OSGi绑定中，选择正确的类加载器并不总是一件简单的事情，Akka的现行做法是每个 `ActorSystem` 实现存储当前线程的上下文类加载器（如果可用，否则就使用他自己的加载器``this.getClass.getClassLoader``），并使用它为所有的反射访问服务。这意味着Akka放在引导类路径（boot class path）下，会从奇怪的地方产生`NullPointerException`： 这里就是不支持。

###应用特定设置
配置也可用于应用程序特定的设置。一个好的实践是将这些设置放在一个扩展中，像下面的章节所描述的：

 * Scala API: [应用特定设置](../chapter6/06_akka_extensions.md#extending-akka-scala-settings)
 * Java API: [应用特定设置](#TODO)

###配置多个ActorSystem
如果你有一个以上的``ActorSystem``（或你正在写一个库，有可能有一个独立于应用的 ``ActorSystem``） 你可能想要为每个系统进行单独配置。

由于 ConfigFactory.load() 会合并classpath中所有匹配名称的资源, 最简单的方式是利用这一功能并在配置树中区分actor系统:

```
  myapp1 {
    akka.loglevel = "WARNING"
    my.own.setting = 43
  }
  myapp2 {
    akka.loglevel = "ERROR"
    app2.setting = "appname"
  }
  my.own.setting = 42
  my.other.setting = "hello"
```

```scala
  val config = ConfigFactory.load()
  val app1 = ActorSystem("MyApp1", config.getConfig("myapp1").withFallback(config))
  val app2 = ActorSystem("MyApp2",
    config.getConfig("myapp2").withOnlyPath("akka").withFallback(config))
```

这两个例子演示了“提升子树”技巧的不同变种: 第一种情况下，actor系统获得的配置是

```ruby
  akka.loglevel = "WARNING"
  my.own.setting = 43
  my.other.setting = "hello"
  // plus myapp1 and myapp2 subtrees
```

而在第二种情况下，只有 “akka” 子树被提升了，结果如下:

```ruby
  akka.loglevel = "ERROR"
  my.own.setting = 42
  my.other.setting = "hello"
  // plus myapp1 and myapp2 subtrees
```

> 注意

> 这个配置文件库非常强大，这里不可能解释其所有的功能。特别是如何在配置文件中包含其它的配置文件 (在[`包含文件Including
  files`](#including-files)中有一个简单的例子) 以及通过路径替换来复制部分配置树。

你也可以在初始化``ActorSystem``时，通过代码的形式，使用其它方法来指定和解析配置信息。

```scala
import akka.actor.ActorSystem
import com.typesafe.config.ConfigFactory
    val customConf = ConfigFactory.parseString("""
      akka.actor.deployment {
        /my-service {
          router = round-robin-pool
          nr-of-instances = 3
        }
      }
      """)
    // ConfigFactory.load sandwiches customConfig between default reference
    // config and default overrides, and then resolves it.
    val system = ActorSystem("MySystem", ConfigFactory.load(customConf))
```

###从自定义位置读取配置
你可以使用代码或系统属性，来替换或补充``application.conf`` 。

如果你使用的方法是``ConfigFactory.load()``（Akka默认方式），你可以通过定义``-Dconfig.resource=whatever``、``-Dconfig.file=whatever``或
``-Dconfig.url=whatever``替换 ``application.conf``。

在``-Dconfig.resource``和相关选项指定的替换配置文件中，如果你还想使用``application.{conf,json,properties}``，可以使用``include
"application"``。在``include
"application"``之前指定的设置会被包含进来的文件内容重写，同理所包含文件的内容也会被之后的内容重写。

在代码中，有很多自定义选项。

``ConfigFactory.load()``有几个重载；这些重载允许你指定夹在 系统属性（重写）和默认值（来自``reference.conf``）之间的配置，并替换通常的``application.{conf,json,properties}``和``-Dconfig.file``相关选项。

``ConfigFactory.load()`` 最简单的变体需要资源基本名称（``application``之外的）；如``myname.conf``、``myname.json``和``myname.properties``而不是``application.{conf,json,properties}``。

最灵活的变体是以一个``Config``对象为参数，你可以使用``ConfigFactory``中的任何方法加载。例如，你可以在代码中使用``ConfigFactory.parseString()``处理一个配置字符串，或者你可以使用``ConfigFactory.parseMap()``创建一个映射，或者也可以加载一个文件。

你也可以将自定义的配置与通常的配置组合起来，像这样：

```java
// make a Config with just your special setting
Config myConfig =
  ConfigFactory.parseString("something=somethingElse");
// load the normal config stack (system props,
// then application.conf, then reference.conf)
Config regularConfig =
  ConfigFactory.load();
// override regular stack with myConfig
Config combined =
  myConfig.withFallback(regularConfig);
// put the result in between the overrides
// (system props) and defaults again
Config complete =
  ConfigFactory.load(combined);
// create ActorSystem
ActorSystem system =
  ActorSystem.create("myname", complete);
```

使用``Config``对象时，请牢记这个蛋糕有三"层"：

- ``ConfigFactory.defaultOverrides()``（系统属性）
- 应用设置
- ``ConfigFactory.defaultReference()``（reference.conf）

正常的目标是要自定义中间一层，不管其他两个。

 - ``ConfigFactory.load()``加载整个堆栈
 - ``ConfigFactory.load()``的重载允许你指定一个不同的中间层
 - ``ConfigFactory.parse()``变体加载单个文件或资源

要叠加两层，可使用``override.withFallback(fallback)``；请努力保持系统属性（``defaultOverrides()``）在顶部，``reference.conf`` （``defaultReference()``）在底部。

要记住，通常你只需要在``application.conf``添加一个``include``语句，而不是编写代码。在``application.conf``顶部引入的将被``application.conf``其余部分覆盖，而那些在底部的设置将覆盖以前的内容。

###Actor部署配置
可以在配置的``akka.actor.deployment``节中定义特定actor的部署设置。在部署部分有可能定义这些事物——调度器、邮箱、路由器设置和远程部署。在相应主题的章节中详细介绍了配置的这些特性。一个例子，可以如下所示：

```
akka.actor.deployment {

  # '/user/actorA/actorB' is a remote deployed actor
  /actorA/actorB {
    remote = "akka.tcp://sampleActorSystem@127.0.0.1:2553"
  }

  # all direct children of '/user/actorC' have a dedicated dispatcher
  "/actorC/*" {
    dispatcher = my-dispatcher
  }

  # '/user/actorD/actorE' has a special priority mailbox
  /actorD/actorE {
    mailbox = prio-mailbox
  }

  # '/user/actorF/actorG/actorH' is a random pool
  /actorF/actorG/actorH {
    router = random-pool
    nr-of-instances = 5
  }
}

my-dispatcher {
  fork-join-executor.parallelism-min = 10
  fork-join-executor.parallelism-max = 10
}
prio-mailbox {
  mailbox-type = "a.b.MyPrioMailbox"
}
```

一个指定actor部署部分的设置是通过其相对 ``/user``的路径来标识的。

你可以使用星号作为通配符匹配actor的路径部分，所以你可以指定：``/*/sampleActor``将匹配该树形结构中那个级别上的所有``sampleActor``。你也能把通配符放在最后来匹配某一级别的所有actor：``/someParent/*``。非通配符匹配总是有更高的优先级，所以：``/foo/bar``比``/foo/*`` **更具体**，并且只有最高优先的匹配才会被使用。请注意它**不能**用于部分匹配，像这样：``/foo*/bar``、``/f*o/bar`` 等。

###参考配置清单

每个Akka模块都有保存默认值的“reference”配置文件。

<span id="config-akka-actor"></span>

#####akka-actor

```
####################################
# Akka Actor Reference Config File #
####################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {
  # Akka version, checked against the runtime version of Akka.
  version = "2.3.6"

  # Home directory of Akka, modules in the deploy directory will be loaded
  home = ""

  # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
  # to STDOUT)
  loggers = ["akka.event.Logging$DefaultLogger"]

  # Loggers are created and registered synchronously during ActorSystem
  # start-up, and since they are actors, this timeout is used to bound the
  # waiting time
  logger-startup-timeout = 5s

  # Log level used by the configured loggers (see "loggers") as soon
  # as they have been started; before that, see "stdout-loglevel"
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "INFO"

  # Log level for the very basic logger activated during ActorSystem startup.
  # This logger prints the log messages to stdout (System.out).
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  stdout-loglevel = "WARNING"

  # Log the complete configuration at INFO level when the actor system is started.
  # This is useful when you are uncertain of what configuration is used.
  log-config-on-start = off

  # Log at info level when messages are sent to dead letters.
  # Possible values:
  # on: all dead letters are logged
  # off: no logging of dead letters
  # n: positive integer, number of dead letters that will be logged
  log-dead-letters = 10

  # Possibility to turn off logging of dead letters while the actor system
  # is shutting down. Logging is only done when enabled by 'log-dead-letters'
  # setting.
  log-dead-letters-during-shutdown = on

  # List FQCN of extensions which shall be loaded at actor system startup.
  # Should be on the format: 'extensions = ["foo", "bar"]' etc.
  # See the Akka Documentation for more info about Extensions
  extensions = []

  # Toggles whether threads created by this ActorSystem should be daemons or not
  daemonic = off

  # JVM shutdown, System.exit(-1), in case of a fatal error,
  # such as OutOfMemoryError
  jvm-exit-on-fatal-error = on

  actor {

    # FQCN of the ActorRefProvider to be used; the below is the built-in default,
    # another one is akka.remote.RemoteActorRefProvider in the akka-remote bundle.
    provider = "akka.actor.LocalActorRefProvider"

    # The guardian "/user" will use this class to obtain its supervisorStrategy.
    # It needs to be a subclass of akka.actor.SupervisorStrategyConfigurator.
    # In addition to the default there is akka.actor.StoppingSupervisorStrategy.
    guardian-supervisor-strategy = "akka.actor.DefaultSupervisorStrategy"

    # Timeout for ActorSystem.actorOf
    creation-timeout = 20s

    # Frequency with which stopping actors are prodded in case they had to be
    # removed from their parents
    reaper-interval = 5s

    # Serializes and deserializes (non-primitive) messages to ensure immutability,
    # this is only intended for testing.
    serialize-messages = off

    # Serializes and deserializes creators (in Props) to ensure that they can be
    # sent over the network, this is only intended for testing. Purely local deployments
    # as marked with deploy.scope == LocalScope are exempt from verification.
    serialize-creators = off

    # Timeout for send operations to top-level actors which are in the process
    # of being started. This is only relevant if using a bounded mailbox or the
    # CallingThreadDispatcher for a top-level actor.
    unstarted-push-timeout = 10s

    typed {
      # Default timeout for typed actor methods with non-void return type
      timeout = 5s
    }

    # Mapping between ´deployment.router' short names to fully qualified class names
    router.type-mapping {
      from-code = "akka.routing.NoRouter"
      round-robin-pool = "akka.routing.RoundRobinPool"
      round-robin-group = "akka.routing.RoundRobinGroup"
      random-pool = "akka.routing.RandomPool"
      random-group = "akka.routing.RandomGroup"
      balancing-pool = "akka.routing.BalancingPool"
      smallest-mailbox-pool = "akka.routing.SmallestMailboxPool"
      broadcast-pool = "akka.routing.BroadcastPool"
      broadcast-group = "akka.routing.BroadcastGroup"
      scatter-gather-pool = "akka.routing.ScatterGatherFirstCompletedPool"
      scatter-gather-group = "akka.routing.ScatterGatherFirstCompletedGroup"
      tail-chopping-pool = "akka.routing.TailChoppingPool"
      tail-chopping-group = "akka.routing.TailChoppingGroup"
      consistent-hashing-pool = "akka.routing.ConsistentHashingPool"
      consistent-hashing-group = "akka.routing.ConsistentHashingGroup"
    }

    deployment {

      # deployment id pattern - on the format: /parent/child etc.
      default {

        # The id of the dispatcher to use for this actor.
        # If undefined or empty the dispatcher specified in code
        # (Props.withDispatcher) is used, or default-dispatcher if not
        # specified at all.
        dispatcher = ""

        # The id of the mailbox to use for this actor.
        # If undefined or empty the default mailbox of the configured dispatcher
        # is used or if there is no mailbox configuration the mailbox specified
        # in code (Props.withMailbox) is used.
        # If there is a mailbox defined in the configured dispatcher then that
        # overrides this setting.
        mailbox = ""

        # routing (load-balance) scheme to use
        # - available: "from-code", "round-robin", "random", "smallest-mailbox",
        #              "scatter-gather", "broadcast"
        # - or:        Fully qualified class name of the router class.
        #              The class must extend akka.routing.CustomRouterConfig and
        #              have a public constructor with com.typesafe.config.Config
        #              and optional akka.actor.DynamicAccess parameter.
        # - default is "from-code";
        # Whether or not an actor is transformed to a Router is decided in code
        # only (Props.withRouter). The type of router can be overridden in the
        # configuration; specifying "from-code" means that the values specified
        # in the code shall be used.
        # In case of routing, the actors to be routed to can be specified
        # in several ways:
        # - nr-of-instances: will create that many children
        # - routees.paths: will route messages to these paths using ActorSelection,
        #   i.e. will not create children
        # - resizer: dynamically resizable number of routees as specified in
        #   resizer below
        router = "from-code"

        # number of children to create in case of a router;
        # this setting is ignored if routees.paths is given
        nr-of-instances = 1

        # within is the timeout used for routers containing future calls
        within = 5 seconds

        # number of virtual nodes per node for consistent-hashing router
        virtual-nodes-factor = 10

        tail-chopping-router {
          # interval is duration between sending message to next routee
          interval = 10 milliseconds
        }

        routees {
          # Alternatively to giving nr-of-instances you can specify the full
          # paths of those actors which should be routed to. This setting takes
          # precedence over nr-of-instances
          paths = []
        }

        # To use a dedicated dispatcher for the routees of the pool you can
        # define the dispatcher configuration inline with the property name
        # 'pool-dispatcher' in the deployment section of the router.
        # For example:
        # pool-dispatcher {
        #   fork-join-executor.parallelism-min = 5
        #   fork-join-executor.parallelism-max = 5
        # }

        # Routers with dynamically resizable number of routees; this feature is
        # enabled by including (parts of) this section in the deployment
        resizer {

          enabled = off

          # The fewest number of routees the router should ever have.
          lower-bound = 1

          # The most number of routees the router should ever have.
          # Must be greater than or equal to lower-bound.
          upper-bound = 10

          # Threshold used to evaluate if a routee is considered to be busy
          # (under pressure). Implementation depends on this value (default is 1).
          # 0:   number of routees currently processing a message.
          # 1:   number of routees currently processing a message has
          #      some messages in mailbox.
          # > 1: number of routees with at least the configured pressure-threshold
          #      messages in their mailbox. Note that estimating mailbox size of
          #      default UnboundedMailbox is O(N) operation.
          pressure-threshold = 1

          # Percentage to increase capacity whenever all routees are busy.
          # For example, 0.2 would increase 20% (rounded up), i.e. if current
          # capacity is 6 it will request an increase of 2 more routees.
          rampup-rate = 0.2

          # Minimum fraction of busy routees before backing off.
          # For example, if this is 0.3, then we'll remove some routees only when
          # less than 30% of routees are busy, i.e. if current capacity is 10 and
          # 3 are busy then the capacity is unchanged, but if 2 or less are busy
          # the capacity is decreased.
          # Use 0.0 or negative to avoid removal of routees.
          backoff-threshold = 0.3

          # Fraction of routees to be removed when the resizer reaches the
          # backoffThreshold.
          # For example, 0.1 would decrease 10% (rounded up), i.e. if current
          # capacity is 9 it will request an decrease of 1 routee.
          backoff-rate = 0.1

          # Number of messages between resize operation.
          # Use 1 to resize before each message.
          messages-per-resize = 10
        }
      }
    }

    default-dispatcher {
      # Must be one of the following
      # Dispatcher, PinnedDispatcher, or a FQCN to a class inheriting
      # MessageDispatcherConfigurator with a public constructor with
      # both com.typesafe.config.Config parameter and
      # akka.dispatch.DispatcherPrerequisites parameters.
      # PinnedDispatcher must be used together with executor=thread-pool-executor.
      type = "Dispatcher"

      # Which kind of ExecutorService to use for this dispatcher
      # Valid options:
      #  - "default-executor" requires a "default-executor" section
      #  - "fork-join-executor" requires a "fork-join-executor" section
      #  - "thread-pool-executor" requires a "thread-pool-executor" section
      #  - A FQCN of a class extending ExecutorServiceConfigurator
      executor = "default-executor"

      # This will be used if you have set "executor = "default-executor"".
      # If an ActorSystem is created with a given ExecutionContext, this
      # ExecutionContext will be used as the default executor for all
      # dispatchers in the ActorSystem configured with
      # executor = "default-executor". Note that "default-executor"
      # is the default value for executor, and therefore used if not
      # specified otherwise. If no ExecutionContext is given,
      # the executor configured in "fallback" will be used.
      default-executor {
        fallback = "fork-join-executor"
      }

      # This will be used if you have set "executor = "fork-join-executor""
      fork-join-executor {
        # Min number of threads to cap factor-based parallelism number to
        parallelism-min = 8

        # The parallelism factor is used to determine thread pool size using the
        # following formula: ceil(available processors * factor). Resulting size
        # is then bounded by the parallelism-min and parallelism-max values.
        parallelism-factor = 3.0

        # Max number of threads to cap factor-based parallelism number to
        parallelism-max = 64
      }

      # This will be used if you have set "executor = "thread-pool-executor""
      thread-pool-executor {
        # Keep alive time for threads
        keep-alive-time = 60s

        # Min number of threads to cap factor-based core number to
        core-pool-size-min = 8

        # The core pool size factor is used to determine thread pool core size
        # using the following formula: ceil(available processors * factor).
        # Resulting size is then bounded by the core-pool-size-min and
        # core-pool-size-max values.
        core-pool-size-factor = 3.0

        # Max number of threads to cap factor-based number to
        core-pool-size-max = 64

        # Minimum number of threads to cap factor-based max number to
        # (if using a bounded task queue)
        max-pool-size-min = 8

        # Max no of threads (if using a bounded task queue) is determined by
        # calculating: ceil(available processors * factor)
        max-pool-size-factor  = 3.0

        # Max number of threads to cap factor-based max number to
        # (if using a  bounded task queue)
        max-pool-size-max = 64

        # Specifies the bounded capacity of the task queue (< 1 == unbounded)
        task-queue-size = -1

        # Specifies which type of task queue will be used, can be "array" or
        # "linked" (default)
        task-queue-type = "linked"

        # Allow core threads to time out
        allow-core-timeout = on
      }

      # How long time the dispatcher will wait for new actors until it shuts down
      shutdown-timeout = 1s

      # Throughput defines the number of messages that are processed in a batch
      # before the thread is returned to the pool. Set to 1 for as fair as possible.
      throughput = 5

      # Throughput deadline for Dispatcher, set to 0 or negative for no deadline
      throughput-deadline-time = 0ms

      # For BalancingDispatcher: If the balancing dispatcher should attempt to
      # schedule idle actors using the same dispatcher when a message comes in,
      # and the dispatchers ExecutorService is not fully busy already.
      attempt-teamwork = on

      # If this dispatcher requires a specific type of mailbox, specify the
      # fully-qualified class name here; the actually created mailbox will
      # be a subtype of this type. The empty string signifies no requirement.
      mailbox-requirement = ""
    }

    default-mailbox {
      # FQCN of the MailboxType. The Class of the FQCN must have a public
      # constructor with
      # (akka.actor.ActorSystem.Settings, com.typesafe.config.Config) parameters.
      mailbox-type = "akka.dispatch.UnboundedMailbox"

      # If the mailbox is bounded then it uses this setting to determine its
      # capacity. The provided value must be positive.
      # NOTICE:
      # Up to version 2.1 the mailbox type was determined based on this setting;
      # this is no longer the case, the type must explicitly be a bounded mailbox.
      mailbox-capacity = 1000

      # If the mailbox is bounded then this is the timeout for enqueueing
      # in case the mailbox is full. Negative values signify infinite
      # timeout, which should be avoided as it bears the risk of dead-lock.
      mailbox-push-timeout-time = 10s

      # For Actor with Stash: The default capacity of the stash.
      # If negative (or zero) then an unbounded stash is used (default)
      # If positive then a bounded stash is used and the capacity is set using
      # the property
      stash-capacity = -1
    }

    mailbox {
      # Mapping between message queue semantics and mailbox configurations.
      # Used by akka.dispatch.RequiresMessageQueue[T] to enforce different
      # mailbox types on actors.
      # If your Actor implements RequiresMessageQueue[T], then when you create
      # an instance of that actor its mailbox type will be decided by looking
      # up a mailbox configuration via T in this mapping
      requirements {
        "akka.dispatch.UnboundedMessageQueueSemantics" =
          akka.actor.mailbox.unbounded-queue-based
        "akka.dispatch.BoundedMessageQueueSemantics" =
          akka.actor.mailbox.bounded-queue-based
        "akka.dispatch.DequeBasedMessageQueueSemantics" =
          akka.actor.mailbox.unbounded-deque-based
        "akka.dispatch.UnboundedDequeBasedMessageQueueSemantics" =
          akka.actor.mailbox.unbounded-deque-based
        "akka.dispatch.BoundedDequeBasedMessageQueueSemantics" =
          akka.actor.mailbox.bounded-deque-based
        "akka.dispatch.MultipleConsumerSemantics" =
          akka.actor.mailbox.unbounded-queue-based
      }

      unbounded-queue-based {
        # FQCN of the MailboxType, The Class of the FQCN must have a public
        # constructor with (akka.actor.ActorSystem.Settings,
        # com.typesafe.config.Config) parameters.
        mailbox-type = "akka.dispatch.UnboundedMailbox"
      }

      bounded-queue-based {
        # FQCN of the MailboxType, The Class of the FQCN must have a public
        # constructor with (akka.actor.ActorSystem.Settings,
        # com.typesafe.config.Config) parameters.
        mailbox-type = "akka.dispatch.BoundedMailbox"
      }

      unbounded-deque-based {
        # FQCN of the MailboxType, The Class of the FQCN must have a public
        # constructor with (akka.actor.ActorSystem.Settings,
        # com.typesafe.config.Config) parameters.
        mailbox-type = "akka.dispatch.UnboundedDequeBasedMailbox"
      }

      bounded-deque-based {
        # FQCN of the MailboxType, The Class of the FQCN must have a public
        # constructor with (akka.actor.ActorSystem.Settings,
        # com.typesafe.config.Config) parameters.
        mailbox-type = "akka.dispatch.BoundedDequeBasedMailbox"
      }
    }

    debug {
      # enable function of Actor.loggable(), which is to log any received message
      # at DEBUG level, see the “Testing Actor Systems” section of the Akka
      # Documentation at http://akka.io/docs
      receive = off

      # enable DEBUG logging of all AutoReceiveMessages (Kill, PoisonPill et.c.)
      autoreceive = off

      # enable DEBUG logging of actor lifecycle changes
      lifecycle = off

      # enable DEBUG logging of all LoggingFSMs for events, transitions and timers
      fsm = off

      # enable DEBUG logging of subscription changes on the eventStream
      event-stream = off

      # enable DEBUG logging of unhandled messages
      unhandled = off

      # enable WARN logging of misconfigured routers
      router-misconfiguration = off
    }

    # Entries for pluggable serializers and their bindings.
    serializers {
      java = "akka.serialization.JavaSerializer"
      bytes = "akka.serialization.ByteArraySerializer"
    }

    # Class to Serializer binding. You only need to specify the name of an
    # interface or abstract base class of the messages. In case of ambiguity it
    # is using the most specific configured class, or giving a warning and
    # choosing the “first” one.
    #
    # To disable one of the default serializers, assign its class to "none", like
    # "java.io.Serializable" = none
    serialization-bindings {
      "[B" = bytes
      "java.io.Serializable" = java
    }

    # Configuration items which are used by the akka.actor.ActorDSL._ methods
    dsl {
      # Maximum queue size of the actor created by newInbox(); this protects
      # against faulty programs which use select() and consistently miss messages
      inbox-size = 1000

      # Default timeout to assume for operations like Inbox.receive et al
      default-timeout = 5s
    }
  }

  # Used to set the behavior of the scheduler.
  # Changing the default values may change the system behavior drastically so make
  # sure you know what you're doing! See the Scheduler section of the Akka
  # Documentation for more details.
  scheduler {
    # The LightArrayRevolverScheduler is used as the default scheduler in the
    # system. It does not execute the scheduled tasks on exact time, but on every
    # tick, it will run everything that is (over)due. You can increase or decrease
    # the accuracy of the execution timing by specifying smaller or larger tick
    # duration. If you are scheduling a lot of tasks you should consider increasing
    # the ticks per wheel.
    # Note that it might take up to 1 tick to stop the Timer, so setting the
    # tick-duration to a high value will make shutting down the actor system
    # take longer.
    tick-duration = 10ms

    # The timer uses a circular wheel of buckets to store the timer tasks.
    # This should be set such that the majority of scheduled timeouts (for high
    # scheduling frequency) will be shorter than one rotation of the wheel
    # (ticks-per-wheel * ticks-duration)
    # THIS MUST BE A POWER OF TWO!
    ticks-per-wheel = 512

    # This setting selects the timer implementation which shall be loaded at
    # system start-up.
    # The class given here must implement the akka.actor.Scheduler interface
    # and offer a public constructor which takes three arguments:
    #  1) com.typesafe.config.Config
    #  2) akka.event.LoggingAdapter
    #  3) java.util.concurrent.ThreadFactory
    implementation = akka.actor.LightArrayRevolverScheduler

    # When shutting down the scheduler, there will typically be a thread which
    # needs to be stopped, and this timeout determines how long to wait for
    # that to happen. In case of timeout the shutdown of the actor system will
    # proceed without running possibly still enqueued tasks.
    shutdown-timeout = 5s
  }

  io {

    # By default the select loops run on dedicated threads, hence using a
    # PinnedDispatcher
    pinned-dispatcher {
      type = "PinnedDispatcher"
      executor = "thread-pool-executor"
      thread-pool-executor.allow-core-pool-timeout = off
    }

    tcp {

      # The number of selectors to stripe the served channels over; each of
      # these will use one select loop on the selector-dispatcher.
      nr-of-selectors = 1

      # Maximum number of open channels supported by this TCP module; there is
      # no intrinsic general limit, this setting is meant to enable DoS
      # protection by limiting the number of concurrently connected clients.
      # Also note that this is a "soft" limit; in certain cases the implementation
      # will accept a few connections more or a few less than the number configured
      # here. Must be an integer > 0 or "unlimited".
      max-channels = 256000

      # When trying to assign a new connection to a selector and the chosen
      # selector is at full capacity, retry selector choosing and assignment
      # this many times before giving up
      selector-association-retries = 10

      # The maximum number of connection that are accepted in one go,
      # higher numbers decrease latency, lower numbers increase fairness on
      # the worker-dispatcher
      batch-accept-limit = 10

      # The number of bytes per direct buffer in the pool used to read or write
      # network data from the kernel.
      direct-buffer-size = 128 KiB

      # The maximal number of direct buffers kept in the direct buffer pool for
      # reuse.
      direct-buffer-pool-limit = 1000

      # The duration a connection actor waits for a `Register` message from
      # its commander before aborting the connection.
      register-timeout = 5s

      # The maximum number of bytes delivered by a `Received` message. Before
      # more data is read from the network the connection actor will try to
      # do other work.
      max-received-message-size = unlimited

      # Enable fine grained logging of what goes on inside the implementation.
      # Be aware that this may log more than once per message sent to the actors
      # of the tcp implementation.
      trace-logging = off

      # Fully qualified config path which holds the dispatcher configuration
      # to be used for running the select() calls in the selectors
      selector-dispatcher = "akka.io.pinned-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the read/write worker actors
      worker-dispatcher = "akka.actor.default-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the selector management actors
      management-dispatcher = "akka.actor.default-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # on which file IO tasks are scheduled
      file-io-dispatcher = "akka.actor.default-dispatcher"

      # The maximum number of bytes (or "unlimited") to transfer in one batch
      # when using `WriteFile` command which uses `FileChannel.transferTo` to
      # pipe files to a TCP socket. On some OS like Linux `FileChannel.transferTo`
      # may block for a long time when network IO is faster than file IO.
      # Decreasing the value may improve fairness while increasing may improve
      # throughput.
      file-io-transferTo-limit = 512 KiB

      # The number of times to retry the `finishConnect` call after being notified about
      # OP_CONNECT. Retries are needed if the OP_CONNECT notification doesn't imply that
      # `finishConnect` will succeed, which is the case on Android.
      finish-connect-retries = 5
    }

    udp {

      # The number of selectors to stripe the served channels over; each of
      # these will use one select loop on the selector-dispatcher.
      nr-of-selectors = 1

      # Maximum number of open channels supported by this UDP module Generally
      # UDP does not require a large number of channels, therefore it is
      # recommended to keep this setting low.
      max-channels = 4096

      # The select loop can be used in two modes:
      # - setting "infinite" will select without a timeout, hogging a thread
      # - setting a positive timeout will do a bounded select call,
      #   enabling sharing of a single thread between multiple selectors
      #   (in this case you will have to use a different configuration for the
      #   selector-dispatcher, e.g. using "type=Dispatcher" with size 1)
      # - setting it to zero means polling, i.e. calling selectNow()
      select-timeout = infinite

      # When trying to assign a new connection to a selector and the chosen
      # selector is at full capacity, retry selector choosing and assignment
      # this many times before giving up
      selector-association-retries = 10

      # The maximum number of datagrams that are read in one go,
      # higher numbers decrease latency, lower numbers increase fairness on
      # the worker-dispatcher
      receive-throughput = 3

      # The number of bytes per direct buffer in the pool used to read or write
      # network data from the kernel.
      direct-buffer-size = 128 KiB

      # The maximal number of direct buffers kept in the direct buffer pool for
      # reuse.
      direct-buffer-pool-limit = 1000

      # The maximum number of bytes delivered by a `Received` message. Before
      # more data is read from the network the connection actor will try to
      # do other work.
      received-message-size-limit = unlimited

      # Enable fine grained logging of what goes on inside the implementation.
      # Be aware that this may log more than once per message sent to the actors
      # of the tcp implementation.
      trace-logging = off

      # Fully qualified config path which holds the dispatcher configuration
      # to be used for running the select() calls in the selectors
      selector-dispatcher = "akka.io.pinned-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the read/write worker actors
      worker-dispatcher = "akka.actor.default-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the selector management actors
      management-dispatcher = "akka.actor.default-dispatcher"
    }

    udp-connected {

      # The number of selectors to stripe the served channels over; each of
      # these will use one select loop on the selector-dispatcher.
      nr-of-selectors = 1

      # Maximum number of open channels supported by this UDP module Generally
      # UDP does not require a large number of channels, therefore it is
      # recommended to keep this setting low.
      max-channels = 4096

      # The select loop can be used in two modes:
      # - setting "infinite" will select without a timeout, hogging a thread
      # - setting a positive timeout will do a bounded select call,
      #   enabling sharing of a single thread between multiple selectors
      #   (in this case you will have to use a different configuration for the
      #   selector-dispatcher, e.g. using "type=Dispatcher" with size 1)
      # - setting it to zero means polling, i.e. calling selectNow()
      select-timeout = infinite

      # When trying to assign a new connection to a selector and the chosen
      # selector is at full capacity, retry selector choosing and assignment
      # this many times before giving up
      selector-association-retries = 10

      # The maximum number of datagrams that are read in one go,
      # higher numbers decrease latency, lower numbers increase fairness on
      # the worker-dispatcher
      receive-throughput = 3

      # The number of bytes per direct buffer in the pool used to read or write
      # network data from the kernel.
      direct-buffer-size = 128 KiB

      # The maximal number of direct buffers kept in the direct buffer pool for
      # reuse.
      direct-buffer-pool-limit = 1000

      # The maximum number of bytes delivered by a `Received` message. Before
      # more data is read from the network the connection actor will try to
      # do other work.
      received-message-size-limit = unlimited

      # Enable fine grained logging of what goes on inside the implementation.
      # Be aware that this may log more than once per message sent to the actors
      # of the tcp implementation.
      trace-logging = off

      # Fully qualified config path which holds the dispatcher configuration
      # to be used for running the select() calls in the selectors
      selector-dispatcher = "akka.io.pinned-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the read/write worker actors
      worker-dispatcher = "akka.actor.default-dispatcher"

      # Fully qualified config path which holds the dispatcher configuration
      # for the selector management actors
      management-dispatcher = "akka.actor.default-dispatcher"
    }

  }


}
```

<span id="config-akka-agent"></span>

##### akka-agent


```
####################################
# Akka Agent Reference Config File #
####################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {
  agent {

    # The dispatcher used for agent-send-off actor
    send-off-dispatcher {
      executor = thread-pool-executor
      type = PinnedDispatcher
    }

    # The dispatcher used for agent-alter-off actor
    alter-off-dispatcher {
      executor = thread-pool-executor
      type = PinnedDispatcher
    }
  }
}
```

<span id="config-akka-camel"></span>

#####akka-camel

```
####################################
# Akka Camel Reference Config File #
####################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {
  camel {
    # FQCN of the ContextProvider to be used to create or locate a CamelContext
    # it must implement akka.camel.ContextProvider and have a no-arg constructor
    # the built-in default create a fresh DefaultCamelContext
    context-provider = akka.camel.DefaultContextProvider

    # Whether JMX should be enabled or disabled for the Camel Context
    jmx = off
    # enable/disable streaming cache on the Camel Context
    streamingCache = on
    consumer {
      # Configured setting which determines whether one-way communications
      # between an endpoint and this consumer actor
      # should be auto-acknowledged or application-acknowledged.
      # This flag has only effect when exchange is in-only.
      auto-ack = on

      # When endpoint is out-capable (can produce responses) reply-timeout is the
      # maximum time the endpoint can take to send the response before the message
      # exchange fails. This setting is used for out-capable, in-only,
      # manually acknowledged communication.
      reply-timeout = 1m

      # The duration of time to await activation of an endpoint.
      activation-timeout = 10s
    }

    #Scheme to FQCN mappings for CamelMessage body conversions
    conversions {
      "file" = "java.io.InputStream"
    }
  }
}
```

<span id="config-akka-cluster"></span>

#####akka-cluster

```
######################################
# Akka Cluster Reference Config File #
######################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {

  cluster {
    # Initial contact points of the cluster.
    # The nodes to join automatically at startup.
    # Comma separated full URIs defined by a string on the form of
    # "akka.tcp://system@hostname:port"
    # Leave as empty if the node is supposed to be joined manually.
    seed-nodes = []

    # how long to wait for one of the seed nodes to reply to initial join request
    seed-node-timeout = 5s

    # If a join request fails it will be retried after this period.
    # Disable join retry by specifying "off".
    retry-unsuccessful-join-after = 10s

    # Should the 'leader' in the cluster be allowed to automatically mark
    # unreachable nodes as DOWN after a configured time of unreachability?
    # Using auto-down implies that two separate clusters will automatically be
    # formed in case of network partition.
    # Disable with "off" or specify a duration to enable auto-down.
    auto-down-unreachable-after = off

    # deprecated in 2.3, use 'auto-down-unreachable-after' instead
    auto-down = off

    # The roles of this member. List of strings, e.g. roles = ["A", "B"].
    # The roles are part of the membership information and can be used by
    # routers or other services to distribute work to certain member types,
    # e.g. front-end and back-end nodes.
    roles = []

    role {
      # Minimum required number of members of a certain role before the leader
      # changes member status of 'Joining' members to 'Up'. Typically used together
      # with 'Cluster.registerOnMemberUp' to defer some action, such as starting
      # actors, until the cluster has reached a certain size.
      # E.g. to require 2 nodes with role 'frontend' and 3 nodes with role 'backend':
      #   frontend.min-nr-of-members = 2
      #   backend.min-nr-of-members = 3
      #<role-name>.min-nr-of-members = 1
    }

    # Minimum required number of members before the leader changes member status
    # of 'Joining' members to 'Up'. Typically used together with
    # 'Cluster.registerOnMemberUp' to defer some action, such as starting actors,
    # until the cluster has reached a certain size.
    min-nr-of-members = 1

    # Enable/disable info level logging of cluster events
    log-info = on

    # Enable or disable JMX MBeans for management of the cluster
    jmx.enabled = on

    # how long should the node wait before starting the periodic tasks
    # maintenance tasks?
    periodic-tasks-initial-delay = 1s

    # how often should the node send out gossip information?
    gossip-interval = 1s

    # discard incoming gossip messages if not handled within this duration
    gossip-time-to-live = 2s

    # how often should the leader perform maintenance tasks?
    leader-actions-interval = 1s

    # how often should the node move nodes, marked as unreachable by the failure
    # detector, out of the membership ring?
    unreachable-nodes-reaper-interval = 1s

    # How often the current internal stats should be published.
    # A value of 0s can be used to always publish the stats, when it happens.
    # Disable with "off".
    publish-stats-interval = off

    # The id of the dispatcher to use for cluster actors. If not specified
    # default dispatcher is used.
    # If specified you need to define the settings of the actual dispatcher.
    use-dispatcher = ""

    # Gossip to random node with newer or older state information, if any with
    # this probability. Otherwise Gossip to any random live node.
    # Probability value is between 0.0 and 1.0. 0.0 means never, 1.0 means always.
    gossip-different-view-probability = 0.8

    # Reduced the above probability when the number of nodes in the cluster
    # greater than this value.
    reduce-gossip-different-view-probability = 400

    # Settings for the Phi accrual failure detector (http://ddg.jaist.ac.jp/pub/HDY+04.pdf
    # [Hayashibara et al]) used by the cluster subsystem to detect unreachable
    # members.
    failure-detector {

      # FQCN of the failure detector implementation.
      # It must implement akka.remote.FailureDetector and have
      # a public constructor with a com.typesafe.config.Config and
      # akka.actor.EventStream parameter.
      implementation-class = "akka.remote.PhiAccrualFailureDetector"

      # How often keep-alive heartbeat messages should be sent to each connection.
      heartbeat-interval = 1 s

      # Defines the failure detector threshold.
      # A low threshold is prone to generate many wrong suspicions but ensures
      # a quick detection in the event of a real crash. Conversely, a high
      # threshold generates fewer mistakes but needs more time to detect
      # actual crashes.
      threshold = 8.0

      # Number of the samples of inter-heartbeat arrival times to adaptively
      # calculate the failure timeout for connections.
      max-sample-size = 1000

      # Minimum standard deviation to use for the normal distribution in
      # AccrualFailureDetector. Too low standard deviation might result in
      # too much sensitivity for sudden, but normal, deviations in heartbeat
      # inter arrival times.
      min-std-deviation = 100 ms

      # Number of potentially lost/delayed heartbeats that will be
      # accepted before considering it to be an anomaly.
      # This margin is important to be able to survive sudden, occasional,
      # pauses in heartbeat arrivals, due to for example garbage collect or
      # network drop.
      acceptable-heartbeat-pause = 3 s

      # Number of member nodes that each member will send heartbeat messages to,
      # i.e. each node will be monitored by this number of other nodes.
      monitored-by-nr-of-members = 5

      # After the heartbeat request has been sent the first failure detection
      # will start after this period, even though no heartbeat mesage has
      # been received.
      expected-response-after = 5 s

    }

    metrics {
      # Enable or disable metrics collector for load-balancing nodes.
      enabled = on

      # FQCN of the metrics collector implementation.
      # It must implement akka.cluster.MetricsCollector and
      # have public constructor with akka.actor.ActorSystem parameter.
      # The default SigarMetricsCollector uses JMX and Hyperic SIGAR, if SIGAR
      # is on the classpath, otherwise only JMX.
      collector-class = "akka.cluster.SigarMetricsCollector"

      # How often metrics are sampled on a node.
      # Shorter interval will collect the metrics more often.
      collect-interval = 3s

      # How often a node publishes metrics information.
      gossip-interval = 3s

      # How quickly the exponential weighting of past data is decayed compared to
      # new data. Set lower to increase the bias toward newer values.
      # The relevance of each data sample is halved for every passing half-life
      # duration, i.e. after 4 times the half-life, a data sample’s relevance is
      # reduced to 6% of its original relevance. The initial relevance of a data
      # sample is given by 1 – 0.5 ^ (collect-interval / half-life).
      # See http://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average
      moving-average-half-life = 12s
    }

    # If the tick-duration of the default scheduler is longer than the
    # tick-duration configured here a dedicated scheduler will be used for
    # periodic tasks of the cluster, otherwise the default scheduler is used.
    # See akka.scheduler settings for more details.
    scheduler {
      tick-duration = 33ms
      ticks-per-wheel = 512
    }

  }

  # Default configuration for routers
  actor.deployment.default {
    # MetricsSelector to use
    # - available: "mix", "heap", "cpu", "load"
    # - or: Fully qualified class name of the MetricsSelector class.
    #       The class must extend akka.cluster.routing.MetricsSelector
    #       and have a public constructor with com.typesafe.config.Config
    #       parameter.
    # - default is "mix"
    metrics-selector = mix
  }
  actor.deployment.default.cluster {
    # enable cluster aware router that deploys to nodes in the cluster
    enabled = off

    # Maximum number of routees that will be deployed on each cluster
    # member node.
    # Note that nr-of-instances defines total number of routees, but
    # number of routees per node will not be exceeded, i.e. if you
    # define nr-of-instances = 50 and max-nr-of-instances-per-node = 2
    # it will deploy 2 routees per new member in the cluster, up to
    # 25 members.
    max-nr-of-instances-per-node = 1

    # Defines if routees are allowed to be located on the same node as
    # the head router actor, or only on remote nodes.
    # Useful for master-worker scenario where all routees are remote.
    allow-local-routees = on

    # Deprecated in 2.3, use routees.paths instead
    routees-path = ""

    # Use members with specified role, or all members if undefined or empty.
    use-role = ""

  }

  # Protobuf serializer for cluster messages
  actor {
    serializers {
      akka-cluster = "akka.cluster.protobuf.ClusterMessageSerializer"
    }

    serialization-bindings {
      "akka.cluster.ClusterMessage" = akka-cluster
    }

    router.type-mapping {
      adaptive-pool = "akka.cluster.routing.AdaptiveLoadBalancingPool"
      adaptive-group = "akka.cluster.routing.AdaptiveLoadBalancingGroup"
    }
  }

}
```

<span id="config-akka-multi-node-testkit"></span>

#####akka-multi-node-testkit

```
#############################################
# Akka Remote Testing Reference Config File #
#############################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {
  testconductor {

    # Timeout for joining a barrier: this is the maximum time any participants
    # waits for everybody else to join a named barrier.
    barrier-timeout = 30s

    # Timeout for interrogation of TestConductor’s Controller actor
    query-timeout = 5s

    # Threshold for packet size in time unit above which the failure injector will
    # split the packet and deliver in smaller portions; do not give value smaller
    # than HashedWheelTimer resolution (would not make sense)
    packet-split-threshold = 100ms

    # amount of time for the ClientFSM to wait for the connection to the conductor
    # to be successful
    connect-timeout = 20s

    # Number of connect attempts to be made to the conductor controller
    client-reconnects = 10

    # minimum time interval which is to be inserted between reconnect attempts
    reconnect-backoff = 1s

    netty {
      # (I&O) Used to configure the number of I/O worker threads on server sockets
      server-socket-worker-pool {
        # Min number of threads to cap factor-based number to
        pool-size-min = 1

        # The pool size factor is used to determine thread pool size
        # using the following formula: ceil(available processors * factor).
        # Resulting size is then bounded by the pool-size-min and
        # pool-size-max values.
        pool-size-factor = 1.0

        # Max number of threads to cap factor-based number to
        pool-size-max = 2
      }

      # (I&O) Used to configure the number of I/O worker threads on client sockets
      client-socket-worker-pool {
        # Min number of threads to cap factor-based number to
        pool-size-min = 1

        # The pool size factor is used to determine thread pool size
        # using the following formula: ceil(available processors * factor).
        # Resulting size is then bounded by the pool-size-min and
        # pool-size-max values.
        pool-size-factor = 1.0

        # Max number of threads to cap factor-based number to
        pool-size-max = 2
      }
    }
  }
}
```

<span id="config-akka-persistence"></span>

#####akka-persistence

```
##########################################
# Akka Persistence Reference Config File #
##########################################



akka {

  # Protobuf serialization for persistent messages
  actor {

    serializers {

      akka-persistence-snapshot = "akka.persistence.serialization.SnapshotSerializer"
      akka-persistence-message = "akka.persistence.serialization.MessageSerializer"
    }

    serialization-bindings {

      "akka.persistence.serialization.Snapshot" = akka-persistence-snapshot
      "akka.persistence.serialization.Message" = akka-persistence-message
    }
  }

  persistence {

    journal {

      # Maximum size of a persistent message batch written to the journal.
      max-message-batch-size = 200

      # Maximum size of a confirmation batch written to the journal.
      max-confirmation-batch-size = 10000

      # Maximum size of a deletion batch written to the journal.
      max-deletion-batch-size = 10000

      # Path to the journal plugin to be used
      plugin = "akka.persistence.journal.leveldb"

      # In-memory journal plugin.
      inmem {

        # Class name of the plugin.
        class = "akka.persistence.journal.inmem.InmemJournal"

        # Dispatcher for the plugin actor.
        plugin-dispatcher = "akka.actor.default-dispatcher"
      }

      # LevelDB journal plugin.
      leveldb {

        # Class name of the plugin.
        class = "akka.persistence.journal.leveldb.LeveldbJournal"

        # Dispatcher for the plugin actor.
        plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"

        # Dispatcher for message replay.
        replay-dispatcher = "akka.persistence.dispatchers.default-replay-dispatcher"

        # Storage location of LevelDB files.
        dir = "journal"

        # Use fsync on write
        fsync = on

        # Verify checksum on read.
        checksum = off

        # Native LevelDB (via JNI) or LevelDB Java port
        native = on
      }

      # Shared LevelDB journal plugin (for testing only).
      leveldb-shared {

        # Class name of the plugin.
        class = "akka.persistence.journal.leveldb.SharedLeveldbJournal"

        # Dispatcher for the plugin actor.
        plugin-dispatcher = "akka.actor.default-dispatcher"

        # timeout for async journal operations
        timeout = 10s

        store {

          # Dispatcher for shared store actor.
          store-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"

          # Dispatcher for message replay.
          replay-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"

          # Storage location of LevelDB files.
          dir = "journal"

          # Use fsync on write
          fsync = on

          # Verify checksum on read.
          checksum = off

          # Native LevelDB (via JNI) or LevelDB Java port
          native = on
        }
      }
    }

    snapshot-store {

      # Path to the snapshot store plugin to be used
      plugin = "akka.persistence.snapshot-store.local"

      # Local filesystem snapshot store plugin.
      local {

        # Class name of the plugin.
        class = "akka.persistence.snapshot.local.LocalSnapshotStore"

        # Dispatcher for the plugin actor.
        plugin-dispatcher = "akka.persistence.dispatchers.default-plugin-dispatcher"

        # Dispatcher for streaming snapshot IO.
        stream-dispatcher = "akka.persistence.dispatchers.default-stream-dispatcher"

        # Storage location of snapshot files.
        dir = "snapshots"
      }
    }

    view {

      # Automated incremental view update.
      auto-update = on

      # Interval between incremental updates
      auto-update-interval = 5s

      # Maximum number of messages to replay per incremental view update. Set to
      # -1 for no upper limit.
      auto-update-replay-max = -1
    }

    at-least-once-delivery {
      # Interval between redelivery attempts
      redeliver-interval = 5s

      # After this number of delivery attempts a `ReliableRedelivery.UnconfirmedWarning`
      # message will be sent to the actor.
      warn-after-number-of-unconfirmed-attempts = 5

      # Maximum number of unconfirmed messages that an actor with AtLeastOnceDelivery is
      # allowed to hold in memory.
      max-unconfirmed-messages = 100000
    }

    dispatchers {
      default-plugin-dispatcher {
        type = PinnedDispatcher
        executor = "thread-pool-executor"
      }
      default-replay-dispatcher {
        type = Dispatcher
        executor = "fork-join-executor"
        fork-join-executor {
          parallelism-min = 2
          parallelism-max = 8
        }
      }
      default-stream-dispatcher {
        type = Dispatcher
        executor = "fork-join-executor"
        fork-join-executor {
          parallelism-min = 2
          parallelism-max = 8
        }
      }
    }
  }
}
```

<span id="config-akka-remote"></span>

#####akka-remote

```
#####################################
# Akka Remote Reference Config File #
#####################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

# comments about akka.actor settings left out where they are already in akka-
# actor.jar, because otherwise they would be repeated in config rendering.

akka {

  actor {

    serializers {
      akka-containers = "akka.remote.serialization.MessageContainerSerializer"
      proto = "akka.remote.serialization.ProtobufSerializer"
      daemon-create = "akka.remote.serialization.DaemonMsgCreateSerializer"
    }


    serialization-bindings {
      # Since com.google.protobuf.Message does not extend Serializable but
      # GeneratedMessage does, need to use the more specific one here in order
      # to avoid ambiguity
      "akka.actor.ActorSelectionMessage" = akka-containers
      "com.google.protobuf.GeneratedMessage" = proto
      "akka.remote.DaemonMsgCreate" = daemon-create
    }

    deployment {

      default {

        # if this is set to a valid remote address, the named actor will be
        # deployed at that node e.g. "akka.tcp://sys@host:port"
        remote = ""

        target {

          # A list of hostnames and ports for instantiating the children of a
          # router
          #   The format should be on "akka.tcp://sys@host:port", where:
          #    - sys is the remote actor system name
          #    - hostname can be either hostname or IP address the remote actor
          #      should connect to
          #    - port should be the port for the remote server on the other node
          # The number of actor instances to be spawned is still taken from the
          # nr-of-instances setting as for local routers; the instances will be
          # distributed round-robin among the given nodes.
          nodes = []

        }
      }
    }
  }

  remote {

    ### General settings

    # Timeout after which the startup of the remoting subsystem is considered
    # to be failed. Increase this value if your transport drivers (see the
    # enabled-transports section) need longer time to be loaded.
    startup-timeout = 10 s

    # Timout after which the graceful shutdown of the remoting subsystem is
    # considered to be failed. After the timeout the remoting system is
    # forcefully shut down. Increase this value if your transport drivers
    # (see the enabled-transports section) need longer time to stop properly.
    shutdown-timeout = 10 s

    # Before shutting down the drivers, the remoting subsystem attempts to flush
    # all pending writes. This setting controls the maximum time the remoting is
    # willing to wait before moving on to shut down the drivers.
    flush-wait-on-shutdown = 2 s

    # Reuse inbound connections for outbound messages
    use-passive-connections = on

    # Controls the backoff interval after a refused write is reattempted.
    # (Transports may refuse writes if their internal buffer is full)
    backoff-interval = 5 ms

    # Acknowledgment timeout of management commands sent to the transport stack.
    command-ack-timeout = 30 s

    # If set to a nonempty string remoting will use the given dispatcher for
    # its internal actors otherwise the default dispatcher is used. Please note
    # that since remoting can load arbitrary 3rd party drivers (see
    # "enabled-transport" and "adapters" entries) it is not guaranteed that
    # every module will respect this setting.
    use-dispatcher = "akka.remote.default-remote-dispatcher"

    ### Security settings

    # Enable untrusted mode for full security of server managed actors, prevents
    # system messages to be send by clients, e.g. messages like 'Create',
    # 'Suspend', 'Resume', 'Terminate', 'Supervise', 'Link' etc.
    untrusted-mode = off

    # When 'untrusted-mode=on' inbound actor selections are by default discarded.
    # Actors with paths defined in this white list are granted permission to receive actor
    # selections messages.
    # E.g. trusted-selection-paths = ["/user/receptionist", "/user/namingService"]
    trusted-selection-paths = []

    # Should the remote server require that its peers share the same
    # secure-cookie (defined in the 'remote' section)? Secure cookies are passed
    # between during the initial handshake. Connections are refused if the initial
    # message contains a mismatching cookie or the cookie is missing.
    require-cookie = off

    # Generate your own with the script availbale in
    # '$AKKA_HOME/scripts/generate_config_with_secure_cookie.sh' or using
    # 'akka.util.Crypt.generateSecureCookie'
    secure-cookie = ""

    ### Logging

    # If this is "on", Akka will log all inbound messages at DEBUG level,
    # if off then they are not logged
    log-received-messages = off

    # If this is "on", Akka will log all outbound messages at DEBUG level,
    # if off then they are not logged
    log-sent-messages = off

    # Sets the log granularity level at which Akka logs remoting events. This setting
    # can take the values OFF, ERROR, WARNING, INFO, DEBUG, or ON. For compatibility
    # reasons the setting "on" will default to "debug" level. Please note that the effective
    # logging level is still determined by the global logging level of the actor system:
    # for example debug level remoting events will be only logged if the system
    # is running with debug level logging.
    # Failures to deserialize received messages also fall under this flag.
    log-remote-lifecycle-events = on

    # Logging of message types with payload size in bytes larger than
    # this value. Maximum detected size per message type is logged once,
    # with an increase threshold of 10%.
    # By default this feature is turned off. Activate it by setting the property to
    # a value in bytes, such as 1000b. Note that for all messages larger than this
    # limit there will be extra performance and scalability cost.
    log-frame-size-exceeding = off

    # Log warning if the number of messages in the backoff buffer in the endpoint
    # writer exceeds this limit. It can be disabled by setting the value to off.
    log-buffer-size-exceeding = 50000

    ### Failure detection and recovery

    # Settings for the failure detector to monitor connections.
    # For TCP it is not important to have fast failure detection, since
    # most connection failures are captured by TCP itself.
    transport-failure-detector {

      # FQCN of the failure detector implementation.
      # It must implement akka.remote.FailureDetector and have
      # a public constructor with a com.typesafe.config.Config and
      # akka.actor.EventStream parameter.
      implementation-class = "akka.remote.DeadlineFailureDetector"

      # How often keep-alive heartbeat messages should be sent to each connection.
      heartbeat-interval = 4 s

      # Number of potentially lost/delayed heartbeats that will be
      # accepted before considering it to be an anomaly.
      # A margin to the `heartbeat-interval` is important to be able to survive sudden,
      # occasional, pauses in heartbeat arrivals, due to for example garbage collect or
      # network drop.
      acceptable-heartbeat-pause = 20 s
    }

    # Settings for the Phi accrual failure detector (http://ddg.jaist.ac.jp/pub/HDY+04.pdf
    # [Hayashibara et al]) used for remote death watch.
    watch-failure-detector {

      # FQCN of the failure detector implementation.
      # It must implement akka.remote.FailureDetector and have
      # a public constructor with a com.typesafe.config.Config and
      # akka.actor.EventStream parameter.
      implementation-class = "akka.remote.PhiAccrualFailureDetector"

      # How often keep-alive heartbeat messages should be sent to each connection.
      heartbeat-interval = 1 s

      # Defines the failure detector threshold.
      # A low threshold is prone to generate many wrong suspicions but ensures
      # a quick detection in the event of a real crash. Conversely, a high
      # threshold generates fewer mistakes but needs more time to detect
      # actual crashes.
      threshold = 10.0

      # Number of the samples of inter-heartbeat arrival times to adaptively
      # calculate the failure timeout for connections.
      max-sample-size = 200

      # Minimum standard deviation to use for the normal distribution in
      # AccrualFailureDetector. Too low standard deviation might result in
      # too much sensitivity for sudden, but normal, deviations in heartbeat
      # inter arrival times.
      min-std-deviation = 100 ms

      # Number of potentially lost/delayed heartbeats that will be
      # accepted before considering it to be an anomaly.
      # This margin is important to be able to survive sudden, occasional,
      # pauses in heartbeat arrivals, due to for example garbage collect or
      # network drop.
      acceptable-heartbeat-pause = 10 s


      # How often to check for nodes marked as unreachable by the failure
      # detector
      unreachable-nodes-reaper-interval = 1s

      # After the heartbeat request has been sent the first failure detection
      # will start after this period, even though no heartbeat mesage has
      # been received.
      expected-response-after = 3 s

    }

    # After failed to establish an outbound connection, the remoting will mark the
    # address as failed. This configuration option controls how much time should
    # be elapsed before reattempting a new connection. While the address is
    # gated, all messages sent to the address are delivered to dead-letters.
    # Since this setting limits the rate of reconnects setting it to a
    # very short interval (i.e. less than a second) may result in a storm of
    # reconnect attempts.
    retry-gate-closed-for = 5 s

    # After catastrophic communication failures that result in the loss of system
    # messages or after the remote DeathWatch triggers the remote system gets
    # quarantined to prevent inconsistent behavior.
    # This setting controls how long the Quarantine marker will be kept around
    # before being removed to avoid long-term memory leaks.
    # WARNING: DO NOT change this to a small value to re-enable communication with
    # quarantined nodes. Such feature is not supported and any behavior between
    # the affected systems after lifting the quarantine is undefined.
    prune-quarantine-marker-after = 5 d

    # This setting defines the maximum number of unacknowledged system messages
    # allowed for a remote system. If this limit is reached the remote system is
    # declared to be dead and its UID marked as tainted.
    system-message-buffer-size = 1000

    # This setting defines the maximum idle time after an individual
    # acknowledgement for system messages is sent. System message delivery
    # is guaranteed by explicit acknowledgement messages. These acks are
    # piggybacked on ordinary traffic messages. If no traffic is detected
    # during the time period configured here, the remoting will send out
    # an individual ack.
    system-message-ack-piggyback-timeout = 0.3 s

    # This setting defines the time after internal management signals
    # between actors (used for DeathWatch and supervision) that have not been
    # explicitly acknowledged or negatively acknowledged are resent.
    # Messages that were negatively acknowledged are always immediately
    # resent.
    resend-interval = 2 s

    # WARNING: this setting should not be not changed unless all of its consequences
    # are properly understood which assumes experience with remoting internals
    # or expert advice.
    # This setting defines the time after redelivery attempts of internal management
    # signals are stopped to a remote system that has been not confirmed to be alive by
    # this system before.
    initial-system-message-delivery-timeout = 3 m

    ### Transports and adapters

    # List of the transport drivers that will be loaded by the remoting.
    # A list of fully qualified config paths must be provided where
    # the given configuration path contains a transport-class key
    # pointing to an implementation class of the Transport interface.
    # If multiple transports are provided, the address of the first
    # one will be used as a default address.
    enabled-transports = ["akka.remote.netty.tcp"]

    # Transport drivers can be augmented with adapters by adding their
    # name to the applied-adapters setting in the configuration of a
    # transport. The available adapters should be configured in this
    # section by providing a name, and the fully qualified name of
    # their corresponding implementation. The class given here
    # must implement akka.akka.remote.transport.TransportAdapterProvider
    # and have public constructor without parameters.
    adapters {
      gremlin = "akka.remote.transport.FailureInjectorProvider"
      trttl = "akka.remote.transport.ThrottlerProvider"
    }

    ### Default configuration for the Netty based transport drivers

    netty.tcp {
      # The class given here must implement the akka.remote.transport.Transport
      # interface and offer a public constructor which takes two arguments:
      #  1) akka.actor.ExtendedActorSystem
      #  2) com.typesafe.config.Config
      transport-class = "akka.remote.transport.netty.NettyTransport"

      # Transport drivers can be augmented with adapters by adding their
      # name to the applied-adapters list. The last adapter in the
      # list is the adapter immediately above the driver, while
      # the first one is the top of the stack below the standard
      # Akka protocol
      applied-adapters = []

      transport-protocol = tcp

      # The default remote server port clients should connect to.
      # Default is 2552 (AKKA), use 0 if you want a random available port
      # This port needs to be unique for each actor system on the same machine.
      port = 2552

      # The hostname or ip to bind the remoting to,
      # InetAddress.getLocalHost.getHostAddress is used if empty
      hostname = ""

      # Enables SSL support on this transport
      enable-ssl = false

      # Sets the connectTimeoutMillis of all outbound connections,
      # i.e. how long a connect may take until it is timed out
      connection-timeout = 15 s

      # If set to "<id.of.dispatcher>" then the specified dispatcher
      # will be used to accept inbound connections, and perform IO. If "" then
      # dedicated threads will be used.
      # Please note that the Netty driver only uses this configuration and does
      # not read the "akka.remote.use-dispatcher" entry. Instead it has to be
      # configured manually to point to the same dispatcher if needed.
      use-dispatcher-for-io = ""

      # Sets the high water mark for the in and outbound sockets,
      # set to 0b for platform default
      write-buffer-high-water-mark = 0b

      # Sets the low water mark for the in and outbound sockets,
      # set to 0b for platform default
      write-buffer-low-water-mark = 0b

      # Sets the send buffer size of the Sockets,
      # set to 0b for platform default
      send-buffer-size = 256000b

      # Sets the receive buffer size of the Sockets,
      # set to 0b for platform default
      receive-buffer-size = 256000b

      # Maximum message size the transport will accept, but at least
      # 32000 bytes.
      # Please note that UDP does not support arbitrary large datagrams,
      # so this setting has to be chosen carefully when using UDP.
      # Both send-buffer-size and receive-buffer-size settings has to
      # be adjusted to be able to buffer messages of maximum size.
      maximum-frame-size = 128000b

      # Sets the size of the connection backlog
      backlog = 4096

      # Enables the TCP_NODELAY flag, i.e. disables Nagle’s algorithm
      tcp-nodelay = on

      # Enables TCP Keepalive, subject to the O/S kernel’s configuration
      tcp-keepalive = on

      # Enables SO_REUSEADDR, which determines when an ActorSystem can open
      # the specified listen port (the meaning differs between *nix and Windows)
      # Valid values are "on", "off" and "off-for-windows"
      # due to the following Windows bug: http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=4476378
      # "off-for-windows" of course means that it's "on" for all other platforms
      tcp-reuse-addr = off-for-windows

      # Used to configure the number of I/O worker threads on server sockets
      server-socket-worker-pool {
        # Min number of threads to cap factor-based number to
        pool-size-min = 2

        # The pool size factor is used to determine thread pool size
        # using the following formula: ceil(available processors * factor).
        # Resulting size is then bounded by the pool-size-min and
        # pool-size-max values.
        pool-size-factor = 1.0

        # Max number of threads to cap factor-based number to
        pool-size-max = 2
      }

      # Used to configure the number of I/O worker threads on client sockets
      client-socket-worker-pool {
        # Min number of threads to cap factor-based number to
        pool-size-min = 2

        # The pool size factor is used to determine thread pool size
        # using the following formula: ceil(available processors * factor).
        # Resulting size is then bounded by the pool-size-min and
        # pool-size-max values.
        pool-size-factor = 1.0

        # Max number of threads to cap factor-based number to
        pool-size-max = 2
      }


    }

    netty.udp = ${akka.remote.netty.tcp}
    netty.udp {
      transport-protocol = udp
    }

    netty.ssl = ${akka.remote.netty.tcp}
    netty.ssl = {
      # Enable SSL/TLS encryption.
      # This must be enabled on both the client and server to work.
      enable-ssl = true

      security {
        # This is the Java Key Store used by the server connection
        key-store = "keystore"

        # This password is used for decrypting the key store
        key-store-password = "changeme"

        # This password is used for decrypting the key
        key-password = "changeme"

        # This is the Java Key Store used by the client connection
        trust-store = "truststore"

        # This password is used for decrypting the trust store
        trust-store-password = "changeme"

        # Protocol to use for SSL encryption, choose from:
        # Java 6 & 7:
        #   'SSLv3', 'TLSv1'
        # Java 7:
        #   'TLSv1.1', 'TLSv1.2'
        protocol = "TLSv1"

        # Example: ["TLS_RSA_WITH_AES_128_CBC_SHA", "TLS_RSA_WITH_AES_256_CBC_SHA"]
        # You need to install the JCE Unlimited Strength Jurisdiction Policy
        # Files to use AES 256.
        # More info here:
        # http://docs.oracle.com/javase/7/docs/technotes/guides/security/SunProviders.html#SunJCEProvider
        enabled-algorithms = ["TLS_RSA_WITH_AES_128_CBC_SHA"]

        # There are three options, in increasing order of security:
        # "" or SecureRandom => (default)
        # "SHA1PRNG" => Can be slow because of blocking issues on Linux
        # "AES128CounterSecureRNG" => fastest startup and based on AES encryption
        # algorithm
        # "AES256CounterSecureRNG"
        # The following use one of 3 possible seed sources, depending on
        # availability: /dev/random, random.org and SecureRandom (provided by Java)
        # "AES128CounterInetRNG"
        # "AES256CounterInetRNG" (Install JCE Unlimited Strength Jurisdiction
        # Policy Files first)
        # Setting a value here may require you to supply the appropriate cipher
        # suite (see enabled-algorithms section above)
        random-number-generator = ""
      }
    }

    ### Default configuration for the failure injector transport adapter

    gremlin {
      # Enable debug logging of the failure injector transport adapter
      debug = off
    }

    ### Default dispatcher for the remoting subsystem

    default-remote-dispatcher {
      type = Dispatcher
      executor = "fork-join-executor"
      fork-join-executor {
        # Min number of threads to cap factor-based parallelism number to
        parallelism-min = 2
        parallelism-max = 2
      }
    }

    backoff-remote-dispatcher {
      type = Dispatcher
      executor = "fork-join-executor"
      fork-join-executor {
        # Min number of threads to cap factor-based parallelism number to
        parallelism-min = 2
        parallelism-max = 2
      }
    }


  }

}
```

<span id="config-akka-testkit"></span>

#####akka-testkit

```
######################################
# Akka Testkit Reference Config File #
######################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {
  test {
    # factor by which to scale timeouts during tests, e.g. to account for shared
    # build system load
    timefactor =  1.0

    # duration of EventFilter.intercept waits after the block is finished until
    # all required messages are received
    filter-leeway = 3s

    # duration to wait in expectMsg and friends outside of within() block
    # by default
    single-expect-default = 3s

    # The timeout that is added as an implicit by DefaultTimeout trait
    default-timeout = 5s

    calling-thread-dispatcher {
      type = akka.testkit.CallingThreadDispatcherConfigurator
    }
  }
}
```

<span id="config-akka-zeromq"></span>

#####akka-zeromq

```
#####################################
# Akka ZeroMQ Reference Config File #
#####################################

# This is the reference config file that contains all the default settings.
# Make your edits/overrides in your application.conf.

akka {

  zeromq {

    # The default timeout for a poll on the actual zeromq socket.
    poll-timeout = 100ms

    # Timeout for creating a new socket
    new-socket-timeout = 5s

    socket-dispatcher {
      # A zeromq socket needs to be pinned to the thread that created it.
      # Changing this value results in weird errors and race conditions within
      # zeromq
      executor = thread-pool-executor
      type = "PinnedDispatcher"
      thread-pool-executor.allow-core-timeout = off
    }
  }
}
```


