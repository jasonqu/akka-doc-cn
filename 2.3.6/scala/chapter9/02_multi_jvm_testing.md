# 多JVM测试

> 注：本节未经校验，如有问题欢迎提issue

同时在多个 Jvm 中运行应用程序 （main方法的对象） 和 ScalaTest 测试的支持。对于需要多个系统相互沟通的集成测试很有用。

### 安装程序
多JVM测试是 sbt 插件，你可以在 http://github.com/typesafehub/sbt-multi-jvm 找到。

你可以作为一个插件添加它，在 project/plugins.sbt 添加以下内容：

    addSbtPlugin ("com.typesafe.sbt"%"sbt 多 jvm"%"0.3.8"）

然后，你可以通过在``build.sbt`` 或 ``project/Build.scala``包含``MultiJvm``和设置添加多JVM 测试。请注意 MultiJvm 测试源代码位于 ``src/multi-jvm/...``而不是在``src/test/...``。

下面是使用 MultiJvm 插件的 sbt 0.13 下的 build.sbt 文件示例：

```scala
import com.typesafe.sbt.SbtMultiJvm
import com.typesafe.sbt.SbtMultiJvm.MultiJvmKeys.MultiJvm

val akkaVersion = "2.3.6"

val project = Project(
  id = "akka-sample-multi-node-scala",
  base = file("."),
  settings = Project.defaultSettings ++ SbtMultiJvm.multiJvmSettings ++ Seq(
    name := "akka-sample-multi-node-scala",
    version := "2.3.6",
    scalaVersion := "2.10.4",
    libraryDependencies ++= Seq(
      "com.typesafe.akka" %% "akka-remote" % akkaVersion,
      "com.typesafe.akka" %% "akka-multi-node-testkit" % akkaVersion,
      "org.scalatest" %% "scalatest" % "2.0" % "test"),
    // make sure that MultiJvm test are compiled by the default test compilation
    compile in MultiJvm <<= (compile in MultiJvm) triggeredBy (compile in Test),
    // disable parallel tests
    parallelExecution in Test := false,
    // make sure that MultiJvm tests are executed by the default test target,
    // and combine the results from ordinary test and multi-jvm tests
    executeTests in Test <<= (executeTests in Test, executeTests in MultiJvm) map {
      case (testResults, multiNodeResults)  =>
        val overall =
          if (testResults.overall.id < multiNodeResults.overall.id)
            multiNodeResults.overall
          else
            testResults.overall
        Tests.Output(overall,
          testResults.events ++ multiNodeResults.events,
          testResults.summaries ++ multiNodeResults.summaries)
    }
  )
) configs (MultiJvm)
```

你可以为分支 Jvm 指定 JVM 选项：

    jvmOptions in MultiJvm := Seq("-Xmx256M")

### 运行测试
多JVM任务是类似于正常的任务：``test``, ``test-only`` 和``run``，但在``multi-jvm``配置下。

所以在Akka中，要运行akka-remote 中的所有多JVM测试（在sbt 提示符中）：

    akka-remote-tests/multi-jvm:test

或可以首先修改``akka-remote-tests``项目，然后运行测试：

    project akka-remote-tests
    multi-jvm:test

使用``test-only``运行单个测试：

    multi-jvm:test-only akka.remote.RandomRoutedRemoteActor

可以列出多个测试名称来运行多个特定的测试。使用sbt的 tab 键可以很容易地完成测试的名称。

也可以通过在测试名称后和``--``包括这些选项，来指定 JVM 选项为``test-only``。举个例子：

    multi-jvm:test-only akka.remote.RandomRoutedRemoteActor -- -Dsome.option=something

### 创建应用程序测试

测试通过一种命名约定被发现并结合起来。MultiJvm 测试源代码位于``src/multi-jvm/...``。测试按以下模式命名：

    {TestName}MultiJvm{NodeName}

也就是，每个测试在其名字中有``MultiJvm``。前面的部分将测试/应用划分在单个``TestName``组下并将一起运行。后面的部分``NodeName``，是为每个分叉的 JVM分配的不同的名称。

因此若要创建名为``Sample``的3-节点测试，你可以如下创建三个应用程序：

```scala
    package sample

    object SampleMultiJvmNode1 {
      def main(args: Array[String]) {
        println("Hello from node 1")
      }
    }

    object SampleMultiJvmNode2 {
      def main(args: Array[String]) {
        println("Hello from node 2")
      }
    }

    object SampleMultiJvmNode3 {
      def main(args: Array[String]) {
        println("Hello from node 3")
      }
    }
```

当你在sbt命令行中调用``multi-jvm:run sample.Sample``，会产生三个 Jvm，分别用于每个节点。看起来会像这样：

    > multi-jvm:run sample.Sample
    ...
    [info] * sample.Sample
    [JVM-1] Hello from node 1
    [JVM-2] Hello from node 2
    [JVM-3] Hello from node 3
    [success] Total time: ...

### 更改默认设置
你可以通过在项目中添加以下配置来更改多JVM测试的源代码目录的名称：

       unmanagedSourceDirectories in MultiJvm <<=
          Seq(baseDirectory(_ / "src/some_directory_here")).join

你可以更改``MultiJvm``标识符。例如，使用 ``multiJvmMarker`` 设置更改它为``ClusterTest`` ：

    multiJvmMarker in MultiJvm := "ClusterTest"

现在，你的测试应该命名为 ``{TestName}ClusterTest{NodeName}``。

### JVM 实例配置
你可以为每个生成的 Jvm定义特定 JVM 选项。通过创建一个以节点的名字命名的带有``.opts``后缀的文件，把它们放在测试的同一个目录中。

例如，为``SampleMultiJvmNode1``提供 JVM 选项``-Dakka.remote.port=9991`` 和``-Xmx256m``，让我们创建三个 ``*.opts`` 文件并向其中添加选项。使用空格分隔多个选项。


``SampleMultiJvmNode1.opts``:

    -Dakka.remote.port=9991 -Xmx256m

``SampleMultiJvmNode2.opts``:

    -Dakka.remote.port=9992 -Xmx256m

``SampleMultiJvmNode3.opts``:

    -Dakka.remote.port=9993 -Xmx256m

### ScalaTest
除了应用程序外，它还支持创建 ScalaTest 测试。要这样做，要如上所述使用相同的命名约定，但创建 ScalaTest 套件，而不是对象的main方法。你需要在类路径上有 ScalaTest。这里是一个类似于上面例子的ScalaTest代码：

    package sample

    import org.scalatest.WordSpec
    import org.scalatest.matchers.MustMatchers

    class SpecMultiJvmNode1 extends WordSpec with MustMatchers {
      "A node" should {
        "be able to say hello" in {
          val message = "Hello from node 1"
          message must be("Hello from node 1")
        }
      }
    }

    class SpecMultiJvmNode2 extends WordSpec with MustMatchers {
      "A node" should {
        "be able to say hello" in {
          val message = "Hello from node 2"
          message must be("Hello from node 2")
        }
      }
    }

你需要在sbt提示中运行``multi-jvm:test-only sample.Spec``来执行这些测试。

### 多节点添加
此外对``SbtMultiJvm``插件有一些补充，以适应[实验模块](../chapter8/experimental_modules.md)一节中描述的[多节点测试](../chapter8/02_multi_node_testing.md)。

