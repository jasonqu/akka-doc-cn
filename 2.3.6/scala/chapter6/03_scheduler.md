# 调度器

> 注：本节未经校验，如有问题欢迎提issue

有时需要设定将来发生的事情，这时该怎么办? `ActorSystem` 就能搞定一切! 在那儿你能找到 `scheduler` 方法，它返回一个 `akka.actor.Scheduler` 实例, 这个实例在每个Actor系统里是唯一的，用来在内部指定一段时间后发生的行为。

请注意定时任务是使用 ActorSystem 的 MessageDispatcher 执行的.

你可以计划向actor发送消息或执行任务 (函数或Runnable). 你会得到一个 ``Cancellable`` 类型的返回值，你可以调用 `cancel` 来取消定时操作的执行。

> 警告

> Akka中使用的``Scheduler``的默认实现是基于根据一个固定的时间表清空的工作桶。它不是在确切的时间执行任务，而是在每个时间刻度，它将运行 （结束） 到期的一切。可以通过``akka.scheduler.tick-duration``配置属性修改默认``Scheduler``的时间精度。

##示例

```scala
import akka.actor.Actor
import akka.actor.Props
import scala.concurrent.duration._

    //Use the system's dispatcher as ExecutionContext
    import system.dispatcher

    //Schedules to send the "foo"-message to the testActor after 50ms
    system.scheduler.scheduleOnce(50 milliseconds, testActor, "foo")
```

```scala
//Schedules a function to be executed (send a message to the testActor) after 50ms
system.scheduler.scheduleOnce(50 milliseconds) {
  testActor ! System.currentTimeMillis
}
```

```scala
val Tick = "tick"
class TickActor extends Actor {
  def receive = {
    case Tick => //Do something
  }
}
val tickActor = system.actorOf(Props(classOf[TickActor], this))
//Use system's dispatcher as ExecutionContext
import system.dispatcher

//This will schedule to send the Tick-message
//to the tickActor after 0ms repeating every 50ms
val cancellable =
  system.scheduler.schedule(0 milliseconds,
    50 milliseconds,
    tickActor,
    Tick)

//This cancels further Ticks to be sent
cancellable.cancel()
```

> 警告

> 如果你计划函数或Runnable实例时应该多加小心，不要关闭(闭合包含)不稳定的引用。在实践中这意味着在Actor实例中，不要在闭包中使用``this``，不直接访问 ``sender()`` 并且不要直接调用actor实例的方法。如果你需要安排一个调用，则安排一个发往``self``的消息 （包含必需的参数），然后在收到消息时再调用方法。

###来自 ``akka.actor.ActorSystem``

```scala
/**
 * Light-weight scheduler for running asynchronous tasks after some deadline
 * in the future. Not terribly precise but cheap.
 */
def scheduler: Scheduler
```

###Scheduler 接口

实际的调度程序实现是在 `ActorSystem` 启动时反射加载的，这意味着通过使用 ``akka.scheduler.implementation`` 配置属性它可能提供一个的不同实现。引用的类必须实现以下接口：

```scala
/**
 * An Akka scheduler service. This one needs one special behavior: if
 * Closeable, it MUST execute all outstanding tasks upon .close() in order
 * to properly shutdown all dispatchers.
 *
 * Furthermore, this timer service MUST throw IllegalStateException if it
 * cannot schedule a task. Once scheduled, the task MUST be executed. If
 * executed upon close(), the task may execute before its timeout.
 *
 * Scheduler implementation are loaded reflectively at ActorSystem start-up
 * with the following constructor arguments:
 *  1) the system’s com.typesafe.config.Config (from system.settings.config)
 *  2) a akka.event.LoggingAdapter
 *  3) a java.util.concurrent.ThreadFactory
 */
trait Scheduler {
  /**
   * Schedules a message to be sent repeatedly with an initial delay and
   * frequency. E.g. if you would like a message to be sent immediately and
   * thereafter every 500ms you would set delay=Duration.Zero and
   * interval=Duration(500, TimeUnit.MILLISECONDS)
   *
   * Java & Scala API
   */
  final def schedule(
    initialDelay: FiniteDuration,
    interval: FiniteDuration,
    receiver: ActorRef,
    message: Any)(implicit executor: ExecutionContext,
                  sender: ActorRef = Actor.noSender): Cancellable =
    schedule(initialDelay, interval, new Runnable {
      def run = {
        receiver ! message
        if (receiver.isTerminated)
          throw new SchedulerException("timer active for terminated actor")
      }
    })

  /**
   * Schedules a function to be run repeatedly with an initial delay and a
   * frequency. E.g. if you would like the function to be run after 2 seconds
   * and thereafter every 100ms you would set delay = Duration(2, TimeUnit.SECONDS)
   * and interval = Duration(100, TimeUnit.MILLISECONDS)
   *
   * Scala API
   */
  final def schedule(
    initialDelay: FiniteDuration,
    interval: FiniteDuration)(f: ⇒ Unit)(
      implicit executor: ExecutionContext): Cancellable =
    schedule(initialDelay, interval, new Runnable { override def run = f })

  /**
   * Schedules a function to be run repeatedly with an initial delay and
   * a frequency. E.g. if you would like the function to be run after 2
   * seconds and thereafter every 100ms you would set delay = Duration(2,
   * TimeUnit.SECONDS) and interval = Duration(100, TimeUnit.MILLISECONDS)
   *
   * Java API
   */
  def schedule(
    initialDelay: FiniteDuration,
    interval: FiniteDuration,
    runnable: Runnable)(implicit executor: ExecutionContext): Cancellable

  /**
   * Schedules a message to be sent once with a delay, i.e. a time period that has
   * to pass before the message is sent.
   *
   * Java & Scala API
   */
  final def scheduleOnce(
    delay: FiniteDuration,
    receiver: ActorRef,
    message: Any)(implicit executor: ExecutionContext,
                  sender: ActorRef = Actor.noSender): Cancellable =
    scheduleOnce(delay, new Runnable {
      override def run = receiver ! message
    })

  /**
   * Schedules a function to be run once with a delay, i.e. a time period that has
   * to pass before the function is run.
   *
   * Scala API
   */
  final def scheduleOnce(delay: FiniteDuration)(f: ⇒ Unit)(
    implicit executor: ExecutionContext): Cancellable =
    scheduleOnce(delay, new Runnable { override def run = f })

  /**
   * Schedules a Runnable to be run once with a delay, i.e. a time period that
   * has to pass before the runnable is executed.
   *
   * Java & Scala API
   */
  def scheduleOnce(
    delay: FiniteDuration,
    runnable: Runnable)(implicit executor: ExecutionContext): Cancellable

  /**
   * The maximum supported task frequency of this scheduler, i.e. the inverse
   * of the minimum time interval between executions of a recurring task, in Hz.
   */
  def maxFrequency: Double

}
```

###Cancellable 接口
它使你可以 取消 计划执行的任务。

警告
它不会中止已经启动的任务的执行。

调度的任务会返回一个 `Cancellable` （或抛出`IllegalStateException`时，如果在调度器关闭后尝试使用）。这允许你取消原定执行的东西。

> 警告

> 这不会中止已经开始执行的任务。检查``cancel``的返回值以检测已计划的任务是被取消，还是（最终）被运行。

```scala
/**
 * Signifies something that can be cancelled
 * There is no strict guarantee that the implementation is thread-safe,
 * but it should be good practice to make it so.
 */
trait Cancellable {
  /**
   * Cancels this Cancellable and returns true if that was successful.
   * If this cancellable was (concurrently) cancelled already, then this method
   * will return false although isCancelled will return true.
   *
   * Java & Scala API
   */
  def cancel(): Boolean

  /**
   * Returns true if and only if this Cancellable has been successfully cancelled
   *
   * Java & Scala API
   */
  def isCancelled: Boolean
}
```


