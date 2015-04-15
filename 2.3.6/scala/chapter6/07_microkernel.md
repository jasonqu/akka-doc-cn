# 微内核

> 注：本节未经校验，如有问题欢迎提issue

Akka微内核的目的是提供一个捆绑机制，以便将 Akka 应用程序作为一个单一有效载荷分发，而不需要在 Java 应用程序服务器中运行，或手动创建一个启动脚本。

Akka 微内核包含在[Akka下载](http://akka.io/downloads)中.

要通过微内核运行应用，你需要创建一个 Bootable 类来处理应用的启动和关闭。下面例子中有介绍。

将你的应用jar包放在 ``deploy`` 目录下，并把依赖放入``lib``目录下以便自动装载并放在类路径中。

要启动内核使用 ``bin`` 目录下的脚本, 将应用启动类传进来.

以下是akka下载包中的一个应用与微内核一起运行的例子. 它可以用以下的命令运行 (在unix系统上):
启动脚本添加 ``config`` 目录作为类路径第一个元素，其次是``lib/*``。它以``akka.kernel.Main``为主类运行 java 并提供引导类作为参数。

（基于 unix 的系统）示例命令：

```
bin/akka sample.kernel.hello.HelloKernel
```

使用 ``Ctrl-C`` 来中断并退出微内核.

在Windows机器上你可以使用 ``bin/akka.bat`` 脚本.

以下是Hello Kernel 示例 (参考 ``HelloKernel``创建一个Bootable 类):

```scala
package sample.kernel.hello

import akka.actor.{ Actor, ActorSystem, Props }
import akka.kernel.Bootable

case object Start

class HelloActor extends Actor {
  val worldActor = context.actorOf(Props[WorldActor])

  def receive = {
    case Start => worldActor ! "Hello"
    case message: String =>
      println("Received message '%s'" format message)
  }
}

class WorldActor extends Actor {
  def receive = {
    case message: String => sender() ! (message.toUpperCase + " world!")
  }
}

class HelloKernel extends Bootable {
  val system = ActorSystem("hellokernel")

  def startup = {
    system.actorOf(Props[HelloActor]) ! Start
  }

  def shutdown = {
    system.shutdown()
  }
}
```

