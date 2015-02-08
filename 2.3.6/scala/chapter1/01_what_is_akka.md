# Akka是什么?

##### 可扩展的实时事务处理

我们相信编写出正确的、具有容错性和可扩展性的并发程序太困难了。这多数是因为使用了错误的工具和错误的抽象级别。Akka就是为了改变这种状况而生的。通过使用Actor模型我们提升了抽象级别，为构建可扩展的、有弹性的响应式并发应用提供了一个更好的平台——详见[《响应式宣言》](http://reactivemanifesto.org/) 。在容错性方面我们采用了“let it crash”（让它崩溃）模型，该模型已经在电信行业构建出“自愈合”的应用和永不停机的系统，取得了巨大成功。Actor还为透明的分布式系统以及真正的可扩展高容错应用的基础进行了抽象。

Akka是开源的，可以通过Apache 2许可获得。

可以从 http://akka.io/downloads/ 下载

请注意所有的代码示例都是可编译的，所以如果你想直接获得源代码，可以查看github的"Akka Docs"子项目——[java](http://github.com/akka/akka/tree/v2.3.6/akka-docs/rst/java/code/docs)和[scala](http://github.com/akka/akka/tree/v2.3.6/akka-docs/rst/scala/code/docs)

### Akka实现了独特的混合模型

##### Actors

Actors为你提供:

* 对并发/并行程序的简单的、高级别的抽象。
* 异步、非阻塞、高性能的事件驱动编程模型。
* 非常轻量的事件驱动处理（1G内存可容纳数百万个actors）。

参阅 [Actors (Scala)](../chapter3/01_actors.md) 和 [Actors (Java)](#TODO)


##### 容错性

* 使用“let-it-crash”语义的监控层次体系。
* 监控层次体系可以跨越多个JVM，从而提供真正的容错系统。
* 非常适合编写永不停机、自愈合的高容错系统。

参阅 [容错性 (Scala)](../chapter3/03_fault_tolerance.md) 和 [容错性 (Java)](#TODO)

##### 位置透明性
Akka的所有元素都为分布式环境而设计：所有actor只通过发送消息进行交互，所有操作都是异步的。

集群支持概览请参阅[Java](#TODO)和[Scala](../chapter5/02_cluster_usage.md)文档相关章节。


##### 持久性

actor接收到的消息可以选择性的被持久化，并在actor启动或重启的时候重放。这使得actor能够恢复其状态，即使是在JVM崩溃或正在迁移到另外节点的情况下。

详情请参阅[Java](#TODO)和[Scala](../chapter3/08_persistence.md)相关章节.

###Scala 和 Java APIs
Akka同时提供 [Scala API](../README.md) 和 [Java API](#TODO)。

###Akka的两种使用方式

以库的形式：在web应用中使用，放到 WEB-INF/lib 中或者作为一个普通的Jar包放进classpath。

以微内核的形式：可以将你的应用放进一个独立的内核。

参阅[用例与部署场景](05_usecase_and_deployment_scenarios.md)了解细节。

###商业支持
Typesafe提供Akka的商业许可，提供开发和产品支持，详见[这里](http://www.typesafe.com/how/subscription)






