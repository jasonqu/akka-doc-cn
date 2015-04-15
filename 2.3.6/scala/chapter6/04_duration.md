# Duration

> 注：本节未经校验，如有问题欢迎提issue

Duration在Akka库中被广泛使用，这代表一个特殊的数据类型——`scala.concurrent.duration.Duration`。这个类型的值可以表示无限(`Duration.Inf`, `Duration.MinusInf`)或有限的时间段，或是`Duration.Undefined`。

###有限与无限
试图将无限的duration转换成一个具体的时间单位，如秒，将引发异常，在编译时有不同类型可用于区分两者：

* `FiniteDuration` 保证是有限的，调用 `toNanos` 和相关方法是安全的
* `Duration`可以是有限或无限的，因此这种类型只用与有限性并不重要的场合 ；这是 `FiniteDuration` 的超类。

###Scala
在Scala，时间段可以通过一个迷你DSL来创建，并支持所有期望的算术操作：

```scala
import scala.concurrent.duration._

val fivesec = 5.seconds
val threemillis = 3.millis
val diff = fivesec - threemillis
assert(diff < fivesec)
val fourmillis = threemillis * 4 / 3 // you cannot write it the other way around
val n = threemillis / (1 millisecond)
```

> 注意

> 如果表达式划定了明显的边界（例如在括号里或在参数列表里），你可以省略"."，但是如果时间单位是一行代码的最后一个词，建议你加上它，否则行末分号推断可能会出错，这取决于下一行是如何开始的。

> Java

> Java提供的语法糖比较少，所以你必须用方法调用来拼出要进行的操作:

```java
import scala.concurrent.duration.Duration;
import scala.concurrent.duration.Deadline;
```

```java
final Duration fivesec = Duration.create(5, "seconds");
final Duration threemillis = Duration.create("3 millis");
final Duration diff = fivesec.minus(threemillis);
assert diff.lt(fivesec);
assert Duration.Zero().lt(Duration.Inf());
```

###Deadline
Duration 有一个兄弟类，名为 `Deadline`, 表示一个持有绝对的时间点的类，并且支持通过计算当前时间到deadline之间的差距来生成Duration。当你想要保持一个总体的期限，而无需记录——自己关注使用时间时，这非常有用：

```scala
val deadline = 10.seconds.fromNow
// do something
val rest = deadline.timeLeft
```

在Java中使用duration来创建dealine：

```java
final Deadline deadline = Duration.create(10, "seconds").fromNow();
final Duration rest = deadline.timeLeft();
```
