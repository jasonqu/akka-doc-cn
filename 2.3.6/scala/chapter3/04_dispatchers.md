# 调度器

Akka ``MessageDispatcher``是维持 Akka Actor “运作”的部分, 可以说它是整个机器的引擎。所有的``MessageDispatcher``实现也同时也是一个``ExecutionContext``，这意味着它们可以用来执行任何代码，例如[Future(Scala)](../chapter4/01_futures.md)。

###缺省派发器
在没有为Actor作配置的情况下，每一个``ActorSystem``将有一个缺省的派发器。该缺省派发器可以被配置，默认是使用指定的``default-executor``的一个``Dispatcher``。如果一个ActorSystem是使用传入的ExecutionContext创建的，则该ExecutionContext将被用作所有派发器的默认执行器（“executor”）。如果没有给定ExecutionContext，则会回退使用``akka.actor.default-dispatcher.default-executor.fallback``指定的执行器。缺省情况下是使用“fork-join-executor”，它在大多数情况下拥有非常好的性能。

###查找一个派发器
派发器实现了`ExecutionContext`接口，因此可以用来运行`Future`调用等。

```scala
// for use with Futures, Scheduler, etc.
implicit val executionContext = system.dispatchers.lookup("my-dispatcher")
```

###为Actor指定派发器
如果你希望为你的``Actor``设置非缺省的派发器，你需要做两件事，首先是配置派发器：

```
my-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "fork-join-executor"
  # Configuration for the fork join pool
  fork-join-executor {
    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 2
    # Parallelism (threads) ... ceil(available processors * factor)
    parallelism-factor = 2.0
    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

以下是另一个使用“thread-pool-executor”的例子：

```
my-thread-pool-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "thread-pool-executor"
  # Configuration for the thread pool
  thread-pool-executor {
    # minimum number of threads to cap factor-based core number to
    core-pool-size-min = 2
    # No of core threads ... ceil(available processors * factor)
    core-pool-size-factor = 2.0
    # maximum number of threads to cap factor-based number to
    core-pool-size-max = 10
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 100
}
```

更多选项，请参阅[配置](../chapter2/09_configuration.md)的缺省派发器(default-dispatcher)一节。

然后可以像往常一样创建actor并在部署配置中定义的调度器。

```scala
import akka.actor.Props
val myActor = context.actorOf(Props[MyActor], "myactor")
```

```
akka.actor.deployment {
  /myactor {
    dispatcher = my-dispatcher
  }
}
```

部署配置的替代方法是在代码中定义调度器。如果你在部署配置中定义``dispatcher``，则实际使用的将是此值，而不是以编程方式提供的参数。

```scala

import akka.actor.Props
val myActor =
  context.actorOf(Props[MyActor].withDispatcher("my-dispatcher"), "myactor1")
```

> 注意

> 你在``withDispatcher``中指定的调度器，和在部署文件指定的``dispatcher``设置其实是配置中的一个路径. 所以在这个例子中它位于配置的顶层，但你可以例如把它放在下面的层次，用“.”来代表子层次，象这样：``"foo.bar.my-dispatcher"``

###派发器的类型

一共有4种类型的消息派发器:

* Dispatcher

  * 这是基于事件的调度器，将一组actor绑定到线程池。如果未指定，则它将被使用为默认调度。

  * 可共享性：无限制

  * 邮箱：任何，为每一个Actor创建一个

  * 使用场景：缺省派发器，Bulkheading

  * 底层驱动：``java.util.concurrent.ExecutorService``
    指定的"executor"可使用"fork-join-executor"、"thread-pool-executor"或一个``akka.dispatcher.ExecutorServiceConfigurator``的FQCN 

* PinnedDispatcher

  * 这个调度器为每一个使用它的actor分配一个独立的线程；即每个actor会有其自己的只有一个线程的线程池。

  * 可共享性: 无

  * 邮箱: 任何，为每个Actor创建一个

  * 使用场景: Bulkheading

  * 底层驱动: 任何``akka.dispatch.ThreadPoolExecutorConfigurator``
    缺省为一个“thread-pool-executor”

* BalancingDispatcher

  * 这是基于事件的调度器，将尝试从繁忙的actor到空闲的actor重新分配工作。

  * 所有actor分享单个邮箱，并从中获取他们的消息。

  * 这里假定所有使用此调度器的actor都可以处理发送到其中一个actor的所有的消息；即actor属于actor池，并且对客户端来说没有保证是哪个actor实例实际上处理某个特定的消息。

  * 可共享性: 仅对同一类型的Actor共享

  * 邮箱: 任何，为所有的Actor创建一个

  * 使用场景: Work-sharing

  * 底层驱动：``java.util.concurrent.ExecutorService``
    指定使用 “executor” 使用 “fork-join-executor”, “thread-pool-executor” 或``akka.dispatcher.ExecutorServiceConfigurator``的全称

  * 请注意**不能**作为一个**路由器调度程序**使用 ``BalancingDispatcher``。（但是你可以使用它为**Routees**）

* CallingThreadDispatcher

  * 该调度器只在当前线程上调用。该调度器不会创建任何新的线程，但它可以被相同的actor从不同的线程同时使用。更多信息和限制，请参阅[CallingThreadDispatcher](09_testing_actor_systems.md#scala-callingthreaddispatcher)。
  * 可共享性: 无限制
  * 邮箱: 任何，每Actor每线程创建一个（需要时）
  * 使用场景: 测试
  * 底层使用: 调用的线程 (duh)

#####更多 dispatcher 配置的例子
配置一个``PinnedDispatcher``：

```
my-pinned-dispatcher {
  executor = "thread-pool-executor"
  type = PinnedDispatcher
}
```

然后使用它:

```scala
val myActor =
  context.actorOf(Props[MyActor].withDispatcher("my-pinned-dispatcher"), "myactor2")
```

注意``thread-pool-executor``配置按上述``my-thread-pool-dispatcher``的调度程序例子并不适用。这是因为每个actor使用``PinnedDispatcher``时，会有其自己的线程池，并且该池将只有一个线程。

注意没有保证随时间推移，**相同**的线程会被使用，由于核心池超时被``PinnedDispatcher``用于在空闲actor的情况下降低资源使用率。总是使用相同的线程需要在``PinnedDispatcher``的配置中添加``thread-pool-executor.allow-core-timeout=off``。


