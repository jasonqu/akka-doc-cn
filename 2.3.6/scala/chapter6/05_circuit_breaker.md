# 线路断路器

> 注：本节未经校验，如有问题欢迎提issue

###为什么使用它们？
线路断路器用于提供稳定性并防止在分布式系统中的级联故障。它们应该结合在远程系统之间的接口使用明智的超时，以防止单个组件的故障拖垮所有组件。

作为一个例子，我们有一个web 应用程序与远程的第三方web服务进行交互。假如第三方已用完了他们的容量，他们的数据库也在高荷载作用下熔化。假设数据库在这种情况下失败，第三方 web 服务用了很长的时间来返回一个错误。这进一步使调用在长一段时间后失败。回到我们的 web 应用程序，用户已注意到其表单提交似乎比需要的使用了更长的时间。当然用户知道要做的是点击刷新按钮，将更多的请求添加到其已在运行的请求中。这最终导致 web 应用程序由于资源枯竭而失败。这将会影响所有用户，甚至是那些没有使用依赖于此第三方 web 服务的功能的。

为 web 服务的调用引入断路器会导致请求开始快速失败，让用户知道有什么地方不对劲，并且他们不需要刷新他们的请求。这还局限失效行为只影响到那些正在使用此第三方功能依赖的用户，因为没有资源枯竭，其他用户不再受影响。电路断路器还可以允许聪明的开发者来标记使用不可用功能的网站部分，或也许在断路器处于打开状态时，显示出一些适当的缓存内容。

Akka库提供名为 `akka.pattern.CircuitBreaker`的断路器实现，具有如下所述的行为。

###他们是做什么工作的？

* 在正常操作期间，断路器处于`Closed`状态：
  * 异常或超过配置的 ``callTimeout`` 的调用增加一个失败计数
  * 成功重置失败计数为零
  * 当失败计数达到 ``maxFailures`` 时，断路器跳入`Open`状态
* 在`Open`状态：
  * 所有调用通过 `CircuitBreakerOpenException` 快速失败
  * 经过配置的 `resetTimeout`，断路器进入`Half-Open`状态
* 在`Half-Open`状态：
  * 第一个调用被允许尝试，而不通过快速失败
  * 其他调用就像在`Open`状态一样快速失败
  * 如果第一次调用成功，断路器是重置回`Closed`状态
  * 如果第一次调用失败，断路器再次跳入`Open`状态并经历另一个完整的`resetTimeout`
* 状态转换监听器：
  * 可以为每个状态条目通过 `onOpen`、 `onClose` 和 `onHalfOpen` 提供回调
  * 这些都在提供的 `ExecutionContext` 中执行。

![](../images/circuit-breaker-states.png)

###例子
#####初始化
下面是如何配置一个`CircuitBreaker`：
  * 最多失败5次
  * 调用超时时间为 10 秒
  * 重置超时时间为 1 分钟

######Scala
```scala
import scala.concurrent.duration._
import akka.pattern.CircuitBreaker
import akka.pattern.pipe
import akka.actor.Actor
import akka.actor.ActorLogging
import scala.concurrent.Future
import akka.event.Logging

class DangerousActor extends Actor with ActorLogging {
  import context.dispatcher

  val breaker =
    new CircuitBreaker(context.system.scheduler,
      maxFailures = 5,
      callTimeout = 10.seconds,
      resetTimeout = 1.minute).onOpen(notifyMeOnOpen())

  def notifyMeOnOpen(): Unit =
    log.warning("My CircuitBreaker is now open, and will not close for one minute")
```

######Java
```java
import akka.actor.UntypedActor;
import scala.concurrent.Future;
import akka.event.LoggingAdapter;
import scala.concurrent.duration.Duration;
import akka.pattern.CircuitBreaker;
import akka.event.Logging;

import static akka.pattern.Patterns.pipe;
import static akka.dispatch.Futures.future;

import java.util.concurrent.Callable;

public class DangerousJavaActor extends UntypedActor {

  private final CircuitBreaker breaker;
  private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

  public DangerousJavaActor() {
    this.breaker = new CircuitBreaker(
      getContext().dispatcher(), getContext().system().scheduler(),
      5, Duration.create(10, "s"), Duration.create(1, "m"))
      .onOpen(new Runnable() {
        public void run() {
          notifyMeOnOpen();
        }
      });
  }

  public void notifyMeOnOpen() {
    log.warning("My CircuitBreaker is now open, and will not close for one minute");
  }
```

#####调用保护
下面是如何将`CircuitBreaker`用于保护一个异步调用，以及一个同步调用：

######Scala
```scala
def dangerousCall: String = "This really isn't that dangerous of a call after all"

def receive = {
  case "is my middle name" =>
    breaker.withCircuitBreaker(Future(dangerousCall)) pipeTo sender()
  case "block for me" =>
    sender() ! breaker.withSyncCircuitBreaker(dangerousCall)
}
```

######Java
```java
public String dangerousCall() {
  return "This really isn't that dangerous of a call after all";
}

@Override
public void onReceive(Object message) {
  if (message instanceof String) {
    String m = (String) message;
    if ("is my middle name".equals(m)) {
      pipe(breaker.callWithCircuitBreaker(
        new Callable<Future<String>>() {
          public Future<String> call() throws Exception {
            return future(
                    new Callable<String>() {
                        public String call() {
                          return dangerousCall();
                        }
                      }, getContext().dispatcher());
          }
        }), getContext().dispatcher()).to(getSender());
    }
    if ("block for me".equals(m)) {
      getSender().tell(breaker
        .callWithSyncCircuitBreaker(
          new Callable<String>() {
            @Override
            public String call() throws Exception {
              return dangerousCall();
            }
          }), getSelf());
    }
  }
}
```

> 注意

> 使用`CircuitBreaker`伴生对象的`apply`或`create`方法将返回在调用者的线程中执行回调的`CircuitBreaker`。如果异步`Future`不必要的时候，这可以是很有用的，例如仅调用同步的API。
