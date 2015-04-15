# 开发指南

> 注：本节未经校验，如有问题欢迎提issue

> 注意

> 首先阅读[Akka贡献者指南](https://github.com/akka/akka/blob/master/CONTRIBUTING.md)。

### 代码风格
Akka代码风格遵循 Scala 风格指南。唯一的例外是块注释的样式：

```scala
  /**
    * Style mandated by "Scala Style Guide"
    */

  /**
   * Style adopted in the Akka codebase
   */
```

Akka使用 ``Scalariform`` 作为构建的一部分的格式化源代码。所以随便修改，然后运行的 ``sbt compile``，它将重新格式化为Akka标准代码。

### 过程

* 请确保你已经签署了Akka CLA，如果没有，在[这里签署](http://www.typesafe.com/contribute/cla)。
* 选择一个ticket，如果没有适合你的就创建一个。
* 首先在功能分支工作。类似``wip-<ticket number>-<descriptive name>-<your username>``这样命名。
* 当你完成后，创建一个 GitHub Pull-Request 到目标分支，并且向Akka邮件列表发送邮件表示你希望它被审阅。
* 审查达成一致后，Akka核心团队将对它进行合并。

### 提交消息
请按照如下指导方针创建公共提交和编写提交消息。

1. 如果你的工作跨越多个本地提交 （例如 ； 如果你在主题分支工作时的安全点提交，或在长时间工作分支做合并/rebase 等）则请**不要**提交所有，而是重写历史，将多个提交挤压到单个大提交中，并在此编写良好的提交消息（如下讨论）。有[一篇好的文章](http://sandofsky.com/blog/git-workflow.html)介绍了如何做到这一点。每个提交都应该是能够用于隔离、cherry picked等。

2. 第一行应该是一个描述性的句子，说明提交在做什么。应该达到只是读这一行就能够充分了解提交做了什么。它**不是**只列出ticket编号、打入"次要修复"或类似的话。在第一行末尾加上以`#`开头的ticket编号。如果提交是一个小的修复程序，则已完成。否则请看下一条。

3. 单行描述之后应该是一个空行，然后跟一个提交细节的枚举列表。

示例：

    Completed replication over BookKeeper based transaction log. Fixes #XXX

      * Details 1
      * Details 2
      * Details 3

### 测试
签入的所有代码都**应该**有测试。所有的测试都是用``ScalaTest`` 和 ``ScalaCheck``编写。

* 命名测试为 **Test.scala**，如果他们不依赖于任何外部的东西。这使得必经之快乐。
* 命名测试为 **Spec.scala**，如果他们具有外部依赖项。

##### Actor测试工具包
测试actor的有用套件：[akka.util.TestKit](http://github.com/akka/akka/tree/v2.3.6/akka-testkit/src/main/scala/akka/testkit/TestKit.scala)。它提供了确认收到的答复及其时间的断言，更多文档参考[测试actor系统模块](../chapter3/09_testing_actor_systems.md)中。

##### 多JVM测试
包含在示例中的是一个多 jvm 测试的 sbt 特质，将分枝 Jvm用于多节点测试。它支持运行应用程序 （含main方法的对象） 和运行 ScalaTest 测试。

##### NetworkFailureTest
你可以使用 'NetworkFailureTest' 特质来测试网络故障。