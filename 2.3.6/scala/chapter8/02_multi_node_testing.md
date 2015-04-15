# 多节点测试

> 注：本节未经校验，如有问题欢迎提issue

### 多节点测试概念

当我们谈论Akka多节点测试时，我们指的是多个actor系统在不同Jvm上运行协同测试的过程。多节点测试套件由三个主要部分组成。

* [`测试导体`](#The_Test_Conductor)。协调和控制测试的节点。
* [`多节点规格`](#The_Multi_Node_Spec)。启动``TestConductor``的方便的包装器，并允许所有节点连接到它。
* [`SbtMultiJvm`](#The_SbtMultiJvm_Plugin)插件。启动可能在多个机器上的多个Jvm的测试。

### <a name="The_Test_Conductor"></a>测试导线
多节点测试的基础是 ``TestConductor``。它是插入到网络栈的一个Akka扩展，它用来协调参与测试的节点，并提供多种功能，包括：

* 节点地址查找： 找出另一个测试节点的完整路径 （不需要在测试节点之间共享配置）
* 节点屏障协调： 在指定屏障等待其他节点。
* 网络故障注入： 流量限制、丢失的数据包、拔出节点并重新插入。

这是测试指挥示意性概述。

.. image:: ../images/akka-remote-testconductor.png

测试导体服务器负责协调屏障，并向对他们采取行动的测试导体客户端发送命令，例如限制从/到另一个客户端的网络流量。有关可能操作的详细信息见 ``akka.remote.testconductor.Conductor`` API 文档。

### <a name="The_Multi_Node_Spec"></a>多节点规范
多节点规范由两部分组成。负责对常见配置，并且枚举和命名测试节点的``MultiNodeConfig``。``MultiNodeSpec``中包含许多方便的函数使测试节点彼此交互。有关可能的操作的详细信息见 ``akka.remote.testkit.MultiNodeSpec`` API 文档。

``MultiNodeSpec`` 的启动设置是通过java 系统属性配置的，你需要对要运行测试的节点的所有Jvm设定的。这些可以轻松地通过在 JVM 命令行中使用``-Dproperty=value``设置。

以下是是可用的属性：

* ``multinode.max-nodes``

  一个测试可以有的节点的最大数目。

* ``multinode.host``

  此节点的主机名或 IP 。必须能够使用 `InetAddress.getByName`解析。

* ``multinode.port``

  此节点端口号。默认值为 0，此时将使用一个随机端口。

* ``multinode.server-host``

  主机名或服务器节点 IP 。必须能够使用 `InetAddress.getByName`解析。

* ``multinode.server-port``

  服务器节点的端口号。默认值为 4711。

* ``multinode.index``

  为测试定义的角色序列中本节点的索引。索引 0 是特别指定为服务器的机器。所有的故障注入和节流必须在这个节点进行。

### <a name="The_SbtMultiJvm_Plugin"></a> SbtMultiJvm 插件

[SbtMultiJvm 插件](multi-jvm-testing.md)已更新以支持运行多节点测试，通过自动生成相关的 ``multinode.*`` 属性。这意味着你可以轻松地在单台计算机上运行多节点测试，而不用任何特殊配置，只要像运行正常多jvm测试一样运行即可。这些测试无需任何更改就可以分布运行在多台机器上，只需使用插件的多节点附加。

##### 多节点特定附加

该插件也有大量的新的``multi-node-*`` sbt 任务和支持多台机器上运行测试的设置。必要的测试类和依赖项通过[SbtAssembly](https://github.com/sbt/sbt-assembly)被打包入一个 jar 文件分发给其他机器，其命名格式为``<projectName>_<scalaVersion>-<projectVersion>-multi-jvm-assembly.jar``。

> 注意

> 要能分发并在多个机器上启动测试，需假定系统主机和目标系统是类 POSIX 系统，并支持 ``ssh`` 和 ``rsync``。

以下都是可用的 sbt 多节点的设置：

* ``multiNodeHosts``

  用于运行测试的主机，以``user@host:java``的形式，其中主机是唯一必须的部分。将覆盖文件中的设置。

* ``multiNodeHostsFileName``

  用于在运行测试的主机读取的文件。与上述格式相同，每行一个。默认为项目基目录中的``multi-node-test.hosts``。

* ``multiNodeTargetDirName``

  目标系统上的目录名称，用于复制 jar 文件。默认为ssh 用户基目录的``multi-node-test``，用于 rsync  jar 文件。

* ``multiNodeJavaName``

  目标机器上的默认 Java 可执行文件的名称。默认值为 ``java``。

这里是如何定义主机的一些示例：

* ``localhost``

  本地主机上的当前用户，使用缺省 java。

* ``user1@host1``

  ``host1``主机上的用户 ``user1``，使用缺省 java。

* ``user2@host2:/usr/lib/jvm/java-7-openjdk-amd64/bin/java``

  ``host2``主机上的用户 ``user2``，使用 java 7。

* ``host3:/usr/lib/jvm/java-6-openjdk-amd64/bin/java``

  ``host3``主机上的当前用户 ``user2``，使用 java 6。

##### 运行多节点测试

若要在多节点模式下从sbt中运行所有的多节点测试 （即分发 jar 文件并开启远程测试），使用``multi-node-test``任务：

    multi-node-test

若要在多 jvm 模式中运行所有测试（即在本地计算机上的所有 Jvm），执行：

    multi-jvm:test

若要运行单个测试使用``multi-node-test-only``任务：

    multi-node-test-only your.MultiNodeTest

若要在多 jvm 模式下运行单个测试，执行：

    multi-jvm:test-only your.MultiNodeTest

可以列出多个测试名称来运行多个特定的测试。sbt的 tab 键提示可以帮你很容易地写出测试的名称。

### 为你的项目准备多节点测试
多节点测试套件是一个单独的 jar 文件。请确保你的项目中包含以下依存关系：

    "com.typesafe.akka" %% "akka-multi-node-testkit" % "2.3.6"

如果你正在使用最新的每夜构建，你应该从“http://repo.typesafe.com/typesafe/snapshots/com/typesafe/akka/akka-multi-node-testkit_2.10/”选择一个带时间戳的Akka版本。我们建议不要使用``SNAPSHOT``以获得稳定的构建。

### 多节点测试示例
首先，我们需要一些脚手架来讲``MultiNodeSpec``与你最喜爱的测试框架钩起来。允许定义一个``STMultiNodeSpec``特质使用 ScalaTest 来启动和停止 ``MultiNodeSpec``。

```scala
package sample.multinode

import org.scalatest.{ BeforeAndAfterAll, WordSpecLike }
import org.scalatest.Matchers
import akka.remote.testkit.MultiNodeSpecCallbacks

/**
 * Hooks up MultiNodeSpec with ScalaTest
 */
trait STMultiNodeSpec extends MultiNodeSpecCallbacks
  with WordSpecLike with Matchers with BeforeAndAfterAll {

  override def beforeAll() = multiNodeSpecBeforeAll()

  override def afterAll() = multiNodeSpecAfterAll()
}
```

然后我们需要定义一个配置。让我们利用两个节点``"node1``和``"node2"``，并称之为 ``MultiNodeSampleConfig``。

```scala
package sample.multinode
import akka.remote.testkit.MultiNodeConfig

object MultiNodeSampleConfig extends MultiNodeConfig {
  val node1 = role("node1")
  val node2 = role("node2")
}
```

然后最终到节点测试代码。启动两个节点，并演示了一个屏障，和一个远程actor消息发送和接收。

```scala
package sample.multinode
import akka.remote.testkit.MultiNodeSpec
import akka.testkit.ImplicitSender
import akka.actor.{ Props, Actor }

class MultiNodeSampleSpecMultiJvmNode1 extends MultiNodeSample
class MultiNodeSampleSpecMultiJvmNode2 extends MultiNodeSample

object MultiNodeSample {
  class Ponger extends Actor {
    def receive = {
      case "ping" => sender() ! "pong"
    }
  }
}

class MultiNodeSample extends MultiNodeSpec(MultiNodeSampleConfig)
  with STMultiNodeSpec with ImplicitSender {

  import MultiNodeSampleConfig._
  import MultiNodeSample._

  def initialParticipants = roles.size

  "A MultiNodeSample" must {

    "wait for all nodes to enter a barrier" in {
      enterBarrier("startup")
    }

    "send to and receive from a remote node" in {
      runOn(node1) {
        enterBarrier("deployed")
        val ponger = system.actorSelection(node(node2) / "user" / "ponger")
        ponger ! "ping"
        expectMsg("pong")
      }

      runOn(node2) {
        system.actorOf(Props[Ponger], "ponger")
        enterBarrier("deployed")
      }

      enterBarrier("finished")
    }
  }
}
```

运行此示例最简单的方法是下载[Typesafe Activator](http://www.typesafe.com/platform/getstarted)并打开名为[Akka Multi-Node Testing Sample with Scala](http://www.typesafe.com/activator/template/akka-sample-multi-node-scala)的教程。

### 需要记住的事情
当写多节点测试时有几件事情要牢记在心，否则你的测试行为可能会出现意想不到的行为。

* 不要关闭第一个节点。第一个节点是控制器，而如果它关闭则你的测试将中断。
* 要能够使用``blackhole``, ``passThrough``和 ``throttle``，你必须激活故障注入器和节流传输适配器，通过在你的 MultiNodeConfig中指定``testTransport(on = true)`` 。
* 限流，关闭和其他失败注解只可以从第一个节点，又即控制器发出。
* 不要再借点关闭后使用 ``node(address)`` 询问节点地址。要在关闭节点之前抓取其地址。
* 不要在主线程之外的其他线程使用 MultiNodeSpec 方法，比如地址查找，屏障进入等。这也意味着你不应该在actor内部或一个计划的任务中使用他们。

### 配置
多节点测试模块的几个配置属性，请参考[配置](../chapter2/09_configuration.md#config-akka-multi-node-testkit)一节。

