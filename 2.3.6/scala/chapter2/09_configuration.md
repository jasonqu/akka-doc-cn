# 配置

你可以开始使用Akka为不需要定义任何配置，因为Akka提供了合理的默认值。不过你可能需要修改设置来更改默认行为或适应特定的运行环境。可以修改的典型设置的例子：

* 日志级别和后端记录器
* 启用远程
* 消息序列化
* 路由器定义
* 调度的调整

Akka使用[Typesafe配置库](https://github.com/typesafehub/config)，这可能也是你自己的应用或库的不错选择，不管用不用Akka。这个库由Java实现，没有外部的依赖；本文后面将只对这个库有一些归纳，你应该查看其文档参考具体的使用（尤其是[ConfigFactory](http://typesafehub.github.io/config/v1.2.0/com/typesafe/config/ConfigFactory.html)）。


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

> 如果您正在编写一个Akka 应用，将你的配置保存类路径的根目录下的``application.conf``文件中。如果您正在编写一个基于Akka的库，将其配置保存在JAR包根目录下的``reference.conf``文件中。

###当使用JarJar,，OneJar，Assembly或任何jar打包命令（jar-bundler）
.. warning::
Akka的配置方法重度依赖于这个理念——每一模块/jar都有它自己的 `reference.conf`文件，所有这些都将会被配置发现并加载。不幸的是，这也意味着如果你放置/合并多个jar到相同的 jar中，您页需要合并所有的`reference.conf`文件。否则所有的默认设置将会丢失，Akka将无法工作。

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

.. _-Dakka.log-config-on-start:
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

###包含文件
有时包含另一个配置文件内容的能力是非常有用，例如假设你有一个``application.conf``包含所有环境独立设置，然后使用特定环境的设置覆写。

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

如果有疑问，您也可以在创建actor系统之前或之后，很容易很方便地检查配置对象：

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

 * Scala API: :ref:`extending-akka-scala.settings`
 * Java API: :ref:`extending-akka-java.settings`

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

> 这个配置文件库非常强大，这里不可能解释其所有的功能. 特别是如何在配置文件中包含其它的配置文件 (在[`包含文件Including
  files`]()中有一个简单的例子) 以及通过路径替换来复制部分配置树。

你也可以在初始化``ActorSystem``时，通过代码的形式，使用其它方法来指定和解析配置信息。

.. includecode:: code/docs/config/ConfigDocSpec.scala
   :include: imports,custom-config

###从自定义位置读取配置
您可以使用代码或系统属性，来替换或补充``application.conf`` 。

如果您使用的``ConfigFactory.load()``（Akka默认方式），您可以通过定义``-Dconfig.resource=whatever``、``-Dconfig.file=whatever``或
``-Dconfig.url=whatever``替换 ``application.conf``。


在``-Dconfig.resource``和相关选项指定的替换配置文件中，如果你还想使用``application.{conf,json,properties}``，可以使用``include
"application"``。在``include
"application"``之前指定的设置会被包含进来的文件内容重写，同理包含的文件的内容也会被之后的内容重写。

在代码中，有很多自定义选项。

``ConfigFactory.load()``有几个重载；这些重载允许您指定夹在 系统属性（重写）和默认值（来自``reference.conf``）之间的配置，并替换通常的``application.{conf,json,properties}``和``-Dconfig.file``相关选项。

The simplest variant of  takes a resource
basename (instead of ``application``); ``myname.conf``,
``myname.json``, and ``myname.properties`` would then be used
instead of ``application.{conf,json,properties}``.

``ConfigFactory.load()`` 最简单的变体需要资源基本名称（``application``之外的）；如``myname.conf``、``myname.json``和``myname.properties``而不是``application.{conf,json,properties}``。

最灵活的变体是以一个``Config``对象为参数，您可以使用``ConfigFactory``中的任何方法加载。例如，您可以在代码中使用``ConfigFactory.parseString()``处理一个配置字符串，或者你可以使用``ConfigFactory.parseMap()``创建一个映射，或者也可以加载一个文件。

您也可以将自定义的配置与通常的配置组合起来，像这样：

.. includecode:: code/docs/config/ConfigDoc.java
   :include: java-custom-config

使用``Config``对象时，请牢记这个蛋糕有三"层"：

- ``ConfigFactory.defaultOverrides()``（系统属性）
- 应用设置
- ``ConfigFactory.defaultReference()``（reference.conf）

正常的目标是要自定义中间一层，不管其他两个。

 - ``ConfigFactory.load()``加载整个堆栈
 - ``ConfigFactory.load()``的重载允许您指定一个不同的中间层
 - ``ConfigFactory.parse()`变体加载单个文件或资源

要叠加两层，可使用``override.withFallback(fallback)``；请努力保持系统属性（``defaultOverrides()``）在顶部，``reference.conf`` （``defaultReference()``）在底部。

要记住，通常你只需要在``application.conf``添加一个``include``语句，而不是编写代码。在``application.conf``顶部引入的将被``application.conf``其余部分覆盖，而那些在底部的设置将覆盖以前的内容。

###Actor部署配置
可以在配置的``akka.actor.deployment``节中定义特定actor的部署设置。在部署部分有可能定义这些事物——调度器、邮箱、路由器设置和远程部署。在相应主题的章节中详细介绍了配置的这些特性。一个例子，可以如下所示：

.. includecode:: code/docs/config/ConfigDocSpec.scala#deployment-section

一个指定actor部署部分的设置是通过其相对 ``/user``的路径来标识的。

您可以使用星号作为通配符匹配actor的路径部分，所以您可以指定：``/*/sampleActor``将匹配该树形结构中那个级别上的所有``sampleActor``。您也能把通配符放在最后来匹配某一级别的所有actor：``/someParent/*``。非通配符匹配总是有更高的优先级，所以：``/foo/bar``比``/foo/*`` **更具体**，并且只有最高优先的匹配才会。请注意它**不能**用于部分匹配，像这样：``/foo*/bar``、``/f*o/bar`` 等。

###参考配置清单

每个Akka模块都有保存默认值的“reference”配置文件。

.. _config-akka-actor:

akka-actor
~~~~~~~~~~

.. literalinclude:: ../../../akka-actor/src/main/resources/reference.conf
   :language: none

.. _config-akka-agent:

akka-agent
~~~~~~~~~~

.. literalinclude:: ../../../akka-agent/src/main/resources/reference.conf
   :language: none

.. _config-akka-camel:

akka-camel
~~~~~~~~~~

.. literalinclude:: ../../../akka-camel/src/main/resources/reference.conf
   :language: none

.. _config-akka-cluster:

akka-cluster
~~~~~~~~~~~~

.. literalinclude:: ../../../akka-cluster/src/main/resources/reference.conf
   :language: none

.. _config-akka-multi-node-testkit:

akka-multi-node-testkit
~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../../akka-multi-node-testkit/src/main/resources/reference.conf
   :language: none

.. _config-akka-persistence:

akka-persistence
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../../akka-persistence/src/main/resources/reference.conf
   :language: none

.. _config-akka-remote:

akka-remote
~~~~~~~~~~~

.. literalinclude:: ../../../akka-remote/src/main/resources/reference.conf
   :language: none

.. _config-akka-testkit:

akka-testkit
~~~~~~~~~~~~

.. literalinclude:: ../../../akka-testkit/src/main/resources/reference.conf
   :language: none

.. _config-akka-zeromq:

akka-zeromq
~~~~~~~~~~~

.. literalinclude:: ../../../akka-zeromq/src/main/resources/reference.conf
   :language: none


