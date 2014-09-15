# 术语，概念

这一章我们将尝试建立通用的术语，对Akka面向的并发、分布式系统等提供一个坚实的讨论基础。请注意，这里的很多术语都没有统一的定义。我们只是希望在Akka文档的范围内给出可用的定义。

###并发 vs. 并行

并发和并行是相关的定义，有一些微小的不同。*并发* 指的是两个或多个任务都有进展，即使他们没有被同时执行。例如可以这样实现：划分出时间片，几个任务交叉执行，尽管时间片的执行是线性的。*并行* 则是指可以真正同时执行。

###异步 vs. 同步

一个方法调用是 *同步* 的，当调用者不能继续处理，除非方法返回一个值或抛出一个异常。另一方面，一个 *异步* 调用允许调用者在调用方法的有限步后能够继续执行，并且该方法的结束可以被额外的机制通知到（也许是一个注册的回调callback，一个Future或一个消息）。

一个同步的API也许会使用阻塞实现同步性，但也不是必须的。一个CPU极为密集的任务也会导致类似阻塞的行为。通常推荐使用非阻塞API，因为它们能确保系统继续处理。Actor本质上是异步的：一个Actor可以在发送消息后继续处理，而不需要等待消息确实被送达。

###非阻塞 vs. 阻塞

如果一个线程的延迟会导致其它一些线程无限期的延迟，我们称之为 *阻塞*。一个很好的例子是资源可以被线程通过互斥锁独占。如果这个线程无限期地占有这个资源（例如不小心进入死循环），其他等待这个资源的线程就无法处理了。相反地，*非阻塞* 意味着没有线程可以无限期的阻塞其他线程。

相比阻塞操作，我们推荐非阻塞的操作，因为很明显这样系统不会因为阻塞操作而不再继续处理。


###死锁 vs. 饥饿 vs. 活锁

当多个参与者互相等待别人达到某个特殊的状态才能继续处理的时候，*死锁* 出现了。因为如果一些参与者不达到特定状态，所有的参与者都不能执行（就像[《第二十二条军规》](http://en.wikipedia.org/wiki/Catch-22)描述的[那样](http://en.wikipedia.org/wiki/Catch-22_\(logic\))），所有相关子系统都停顿了。死锁和阻塞息息相关，因为阻塞使得一个参与者线程可以无限期地推迟其他线程的处理。

在死锁中，没有参与者可以处理，然而相对的 *饥饿* 可能发生，当有些参与者可以不断地处理，而另一些可能不行。一个典型的场景是一个幼稚的调度算法——总是选择高优先级的任务。如果高优先级的任务数量一直足够多，则低优先级的任务永远不会被完成。

*活锁* 和死锁类似，没有参与者可以处理。区别在于与进程进入等待其他进程处理的“冻结”状态不同，参与者不断地变换他们的状态。一个示例场景是两个参与者和两个特殊的资源。他们分别试图获取资源，并且检查是不是另一个参与者也需要这个资源。如果该资源被另一个参与者请求，则它们试图获取另一个资源。在一个很不幸的情况下，也许两个参与者会不停的在两个资源上“跳跃”，永远在谦让而不使用资源。

###竟态条件




We call it a Race condition when an assumption about the ordering of a set of events might be violated by external non-deterministic effects. Race conditions often arise when multiple threads have a shared mutable state, and the operations of thread on the state might be interleaved causing unexpected behavior. While this is a common case, shared state is not necessary to have race conditions. One example could be a client sending unordered packets (e.g UDP datagrams) P1, P2 to a server. As the packets might potentially travel via different network routes, it is possible that the server receives P2 first and P1 afterwards. If the messages contain no information about their sending order it is impossible to determine by the server that they were sent in a different order. Depending on the meaning of the packets this can cause race conditions.


###非阻塞担保（进展条件）
###推荐文献






