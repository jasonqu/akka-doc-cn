# 日志

> 注：本节未经校验，如有问题欢迎提issue

Logging在Akka中不是依赖于一个特定的日志记录后端的。默认日志消息要打印到 STDOUT，但你可以使用插件如SLF4J logger或自己的logger。日志记录是以异步方式执行的，以确保日志记录具有最小的性能影响。Logging一般意味着IO和锁，如果同步执行会减慢你的代码操作。

###<a name="how-to-log"></a>如何记录日志
创建一个 ``LoggingAdapter`` 并使用它的 ``error``, ``warning``, ``info``, 或 ``debug`` 方法, 如下例所示:

```scala
import akka.event.Logging

class MyActor extends Actor {
  val log = Logging(context.system, this)
  override def preStart() = {
    log.debug("Starting")
  }
  override def preRestart(reason: Throwable, message: Option[Any]) {
    log.error(reason, "Restarting due to [{}] when processing [{}]",
      reason.getMessage, message.getOrElse(""))
  }
  def receive = {
    case "test" => log.info("Received test")
    case x      => log.warning("Received unknown message: {}", x)
  }
}
```

方便起见你可以向actor中混入 ``log`` 成员, 而不是像上例一下定义它.

```scala
class MyActor extends Actor with akka.actor.ActorLogging {
  ...
}
```

``Logging``  的第二个参数是这个日志通道的源. 这个源对象以下面的规则转换成字符串:

* 如果它是Actor或ActorRef，则使用它的路径
* 如果是String，就使用它自己
* 如果是类，则使用其simpleName的近似
* 其它的类型，而且当前作用域中又没有隐式的 `LogSource[T]` 实例则会导致编译错误.

日志消息可以包含参数占位符 ``{}``, 如果该日志级别被打开，则占位符会被替换。如果给出的参数数量多过 占位符的数量，则在日志语句的结尾将被追加一个警告 (即在同一级别的同一行）。你可以只传入一个Java数组来为多个占位符分别提供数据:

```scala
val args = Array("The", "brown", "fox", "jumps", 42)
system.log.debug("five parameters: {}, {}, {}, {}, {}", args)
```

日志源的 Java `Class` 也会被包含在生成的 `LogEvent`中. 如果是简单的字符串，它会被替换成一个 “标志” 类 ``akka.event.DummyClassForStringSources`` 以便对这种情况作特殊处理，例如在SLF4j事件监听器中会使用字符串而不是类名来查找要用到的日志记录器实例。

#####死信的日志记录
默认情况下发送到死信的消息会在info级别记录日志。死信的存在并不一定表示有问题，但它可能是，因此他们默认情况下被记录。在几个消息之后此日志记录被关闭，以避免日志泛滥。你可以完全禁用此记录或调整记录多少死信。在系统关闭期间，你很可能看到死信，因为在actor邮箱中的待执行消息会发送到死信。你还可以禁用死信在关闭过程中的日志记录。

    akka {
      log-dead-letters = 10
      log-dead-letters-during-shutdown = on
    }

若要进一步自定义日志记录或为死信采取其他行动，你可以订阅[事件流](event-bus.md#event-stream-scala)。

#####辅助日志选项
Akka 为非常底层的debug提供了一组配置选项，这些主要是为Akka的开发者所用，而非普通用户。

你一定要将日志级别设为 DEBUG 来使用这些选项:

    akka {
      loglevel = "DEBUG"
    }

如果你想知道Akka装载了哪些配置设置，下面这个配置选项非常有用:

```
akka {
  # 在actor系统启动时以INFO级别记录完整的配置
  # 当你不确定使用的是哪个配置时有用
  log-config-on-start = on
}
```

如果你希望记录用户级消息的细节，则使用``akka.event.LoggingReceive``包装你的actor行为，并打开``receive``选项:

    akka {
      actor {
        debug {
          # 打开 LoggingReceive 功能, 以DEBUG级别记录所有接收到的消息
          receive = on
        }
      }
    }

如果你希望记录Actor处理的所有自动接收的消息的细节：

    akka {
      actor {
        debug {
          # 为所有的 AutoReceiveMessages(Kill, PoisonPill 之类) 打开DEBUG日志
          autoreceive = on
        }
      }
    }

如果你希望记录Actor的所有生命周期变化 (重启，死亡等 ) 的细节:

    akka {
      actor {
        debug {
          # 打开actor生命周期变化的DEBUG日志
          lifecycle = on
        }
      }
    }

如果你希望记录所有继承了LoggingFSM的FSM actor的事件、状态转换和计时器的细节:

    akka {
      actor {
        debug {
          # 打开所有 LoggingFSMs 事件、状态转换和计时器的DEBUG日志
          fsm = on
        }
      }
    }

如果你希望监控对 ActorSystem.eventStream 的订阅/取消订阅:

    akka {
      actor {
        debug {
          # 打开eventStream上订阅关系变化的DEBUG日志
          event-stream = on
        }
      }
    }

#####辅助的远程日志选项
如果你希望以DEBUG级别查看所有远程发送的消息: (这些日志是被传输层发送时所记录的，而非actor)

```
akka {
  remote {
    # 如果打开这个选项，Akka将以DEBUG级别记录所有发出的消息， 不打开则不记录
    log-sent-messages = on
  }
}
```

如果你希望以DEBUG级别查看所有接收到的远程消息: (这些日志是被传输层接收时所记录的，而非actor)

```
akka {
  remote {
    # 如果打开这个选项，Akka将以DEBUG级别记录所有接收到的消息， 不打开则不记录
    log-received-messages = on
  }
}
```

如果你希望以INFO级别查看的消息类型与大于指定限制的负载大小 (字节)：

```
    akka {
      remote {
        # Logging of message types with payload size in bytes larger than
        # this value. Maximum detected size per message type is logged once,
        # with an increase threshold of 10%.
        # By default this feature is turned off. Activate it by setting the property to
        # a value in bytes, such as 1000b. Note that for all messages larger than this
        # limit there will be extra performance and scalability cost.
        log-frame-size-exceeding = 1000b
      }
    }
```

同时参阅 TestKit 的日志选项: [跟踪Actor调用](http://doc.akka.io/docs/akka/2.3.6/scala/testing.html#actor-logging-scala).

#####将日志源转换为字符串和类
在运行时将源对象转换成要插入 `LogEvent` 的源字符串和类的规则是使用隐式参数的方式实现的， 因此是完全可配置的: 只需要创建你自己的 `LogSource[T]` 实例并将它放在创建logger的作用域中即可。

```scala
import akka.event.LogSource
import akka.actor.ActorSystem

object MyType {
  implicit val logSource: LogSource[AnyRef] = new LogSource[AnyRef] {
    def genString(o: AnyRef): String = o.getClass.getName
    override def getClazz(o: AnyRef): Class[_] = o.getClass
  }
}

class MyType(system: ActorSystem) {
  import MyType._
  import akka.event.Logging

  val log = Logging(system, this)
}
```

这个例子创建了一个日志源来模拟Java logger的传统用法，该日志源使用原对象的类名作为日志类别。 在这里加入对 getClazz 的重写只是为了作说明，因为它精确地包含了缺省行为。

> 注意

> 你也可以先创建字符串然后将它作为日志源传入，但要知道这时放入 `LogEvent` 中的 `Class[_]` 将是 `akka.event.DummyClassForStringSources`.

> SLF4J 事件监听器对这种情况会特殊处理 (使用实际的字符串来查找logger实例而不是类名), 你在实现自己的日志适配器时可能也会这么做。

#####关闭日志记录
要关闭日志记录，你可以配置日志级别为``OFF``像这样。

```
akka {
  stdout-loglevel = "OFF"
  loglevel = "OFF"
}
```

``stdout-loglevel``将仅在系统启动和关闭时起作用，并且也设置为 ``OFF``，以确保在系统启动或关闭过程中没有日志记录。

###事件处理器
日志记录是通过一个事件总线异步地完成的。日志事件都由一个actor事件处理程序来处理，它将按照它们被发送的相同顺序接收日志事件。

你可以配置的系统启动时创建哪一个事件处理程序来监听日志记录事件。这是使用[配置](../chapter2/09_configuration.md)中的``loggers``元素。你还可以在这里定义日志级别。

```
akka {
  # 在启动时注册的事件处理器 (akka.event.Logging$DefaultLogger 记录日志到标准输出)
  loggers = ["akka.event.Logging$DefaultLogger"]
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "DEBUG"
}
```

缺省会注册一个事件处理器，它将日志记录到标准输出. 在生产系统中不建议使用它。 在 ‘akka-slf4j’ 模块中还有一个 [SLF4J logger](#slf4j-scala)。

创建监听器的示例:

```scala
import akka.event.Logging.InitializeLogger
import akka.event.Logging.LoggerInitialized
import akka.event.Logging.Error
import akka.event.Logging.Warning
import akka.event.Logging.Info
import akka.event.Logging.Debug

class MyEventListener extends Actor {
  def receive = {
    case InitializeLogger(_)                        => sender() ! LoggerInitialized
    case Error(cause, logSource, logClass, message) => // ...
    case Warning(logSource, logClass, message)      => // ...
    case Info(logSource, logClass, message)         => // ...
    case Debug(logSource, logClass, message)        => // ...
  }
}
```

###在启动和关闭时记录日志到stdout
当actor系统启动和关闭配置的``loggers``不会被使用。相反日志消息会打印到标准输出stdout（System.out）。对这个stdout logger的默认的日志级别是``WARNING``，并可以通过设置 ``akka.stdout-loglevel=OFF``使其完全沉默。

###<a name="slf4j-scala"></a>SLF4J
Akka 为 [SL4FJ](http://www.slf4j.org/)提供了一个logger. 它在 ‘akka-slf4j.jar’ 模块中. 它的唯一依赖是 slf4j-api 包. 在运行时，你还需要一个 SLF4J 后端, 我们推荐 [Logback](http://logback.qos.ch/):

```scala
lazy val logback = "ch.qos.logback" % "logback-classic" % "1.0.13"
```

你需要打开[配置](../chapter2/09_configuration.md)中的'loggers'来启用 Slf4jLogger。 你还可以在这里定义事件总线的日志级别。更细粒度的日志级别可以在SLF4j后端的配置（例如：logback.xml）中定义。

```
akka {
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel = "DEBUG"
}
```

一个隐含的迷惑是，时间戳在事件处理程序中标记，而不是实际记录时。

SLF4J 为每个日志事件选择日志记录器是基于创建`LoggingAdapter`时的日志源的 `Class[_]`，除非日志源是一个字符串，那么就使用这个串 (即 ``LoggerFactory.getLogger(c: Class[_]``用于前者而 ``LoggerFactory.getLogger(s: String)`` 用于后者).

> 注意

> 如果创建LoggingAdapter时向工厂方法提供了一个`ActorSystem`，那么该Actor系统的名字将被加在`String`日志源的后面。 如果不希望这样，像下面这样传一个 `LoggingBus` :

```scala
val log = Logging(system.eventStream, "my.nice.string")
```

#####用MDC记录线程日志与Akka源
因为日志的记录是异步的，完成日志记录的线程被保存在 Mapped Diagnostic Context (MDC) 的 ``sourceThread`` 属性里. 在 Logback 的模式布局配置中线程名可以通过 %X{sourceThread} 指定:

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
    <pattern>%date{ISO8601} %-5level %logger{36} %X{sourceThread} - %msg%n</pattern>
  </encoder>
</appender>
```

> 注意

> 最好在应用程序的非Akka的部分也使用MDC的 ``sourceThread`` ，使得这个值在日志里保持一致。

Akka 的另一个有用的工具是创建logger实例时会捕捉 actor 的地址, 这意味着可以访问到actor实例的全部身份信息来将日志信息与其它信息进行关联（例如：路由的成员）。 这个信息保存在MDC的 ``akkaSource``属性中:

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
    <pattern>%date{ISO8601} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
  </encoder>
</appender>
```

要了解这个属性所包括的内容（也适用于非actor）的细节见 [如何记录日志](#how-to-log).

#####MDC中日志输出更准确的时间戳
Akka的日志记录是异步的，这意味着日志条目的时间戳取的是调用底层的记录器实现的时间，这一开始是令人惊讶的。如果你想要更准确地输出时间戳，使用 MDC 属性 ``akkaTimestamp``：

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
    <pattern>%X{akkaTimestamp} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
  </encoder>
</appender>
```

#####由应用程序定义的 MDC 值
Slf4j一个很有用的功能是[MDC](http://logback.qos.ch/manual/mdc.html)，Akka有一种方法让应用程序指定自定义值，你只需要湖区特殊的`LoggingAdapter`——`DiagnosticLoggingAdapter`。为了得到它，你将使用工厂接收actor作为logSource：

```scala
// Within your Actor
val log: DiagnosticLoggingAdapter = Logging(this);
```

一旦你有了记录器，你只需要在记录日志之前先添加自定义的值。这种方式下，这些值将在日志追加前放在 SLF4J MDC中，并在追加后移除。

> 注

> 清理 （清除） 应该在actor之后处理，否则，如果它不设置新的map, 下一条消息海将使用相同的 mdc 值。使用 ``log.clearMDC()``。

```scala
val mdc = Map("requestId" -> 1234, "visitorId" -> 5678)
log.mdc(mdc)

// Log something
log.info("Starting new request")

log.clearMDC()
```

方便起见你可以在actor中混入``log``成员，而不是如上定义。这一特质还允许你重写``def mdc(msg: Any): MDC``用于为当前消息指定 MDC值，并允许你忘记清理，因为它已经为你做过了。

```scala
import Logging.MDC

case class Req(work: String, visitorId: Int)

class MdcActorMixin extends Actor with akka.actor.DiagnosticActorLogging {
  var reqId = 0

  override def mdc(currentMessage: Any): MDC = {
    reqId += 1
    val always = Map("requestId" -> reqId)
    val perMessage = currentMessage match {
      case r: Req => Map("visitorId" -> r.visitorId)
      case _      => Map()
    }
    always ++ perMessage
  }

  def receive: Receive = {
    case r: Req => {
      log.info(s"Starting new request: ${r.work}")
    }
  }
}
```

现在，这些值将可用在MDC中，因此你可以在布局模式中使用它们：

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
    <pattern>
      %-5level %logger{36} [req: %X{requestId}, visitor: %X{visitorId}] - %msg%n
    </pattern>
  </encoder>
</appender>
```


