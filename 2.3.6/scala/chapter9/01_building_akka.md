# 构建Akka

> 注：本节未经校验，如有问题欢迎提issue

此页介绍如何从最新的源代码构建和运行Akka。

### 获取源代码

Akka使用 [Git](http://git-scm.com) 并托管于[Github](http://github.com)。

你首先需要在你的计算机上安装了 Git。然后你可以从http://github.com/akka/akka源代码存储库克隆。

举个例子：

    git clone git://github.com/akka/akka.git

如果你以前已经被克隆存储库，则你可以使用``git pull``更新代码：

### sbt-简单生成工具
Akka使用优秀的 [sbt](https://github.com/harrah/xsbt) 构建系统。所以你要做的第一件事是要下载并安装sbt。你可以在 [sbt 安装文档](https://github.com/harrah/xsbt/wiki/Setup)阅读更多。

所有你需要构建Akka的 sbt 命令会在下面介绍。如果你想要了解更多有关 sbt 的内容，并在自己项目中使用它，请阅读 [sbt 文档](https://github.com/harrah/xsbt/wiki)。

Akka sbt 构建文件是 ``project/AkkaBuild.scala``。

### 构建Akka
首先请确保你在Akka代码目录中：

    cd akka

##### 构建
要编译的Akka核心模块使用``compile``命令：

    sbt compile

可以使用``test``命令来运行所有测试：

    sbt test

如果编译和测试成功然，则你就会拥有一份可以工作的Akka最新的开发版本。

##### 并行执行

默认情况下测试按顺序执行。他们可以并行执行来减少构建时间，如果硬件能控制更大的内存和 cpu 使用率。将以下系统属性添加到sbt的启动脚本来激活并行执行：

    -Dakka.parallelExecution=true

##### 长时间运行和时间敏感试验
默认情况下长时间运行的测试（主要是群集测试）和时间敏感测试 （取决于运行它的机器的性能） 是禁用的。你可以通过添加以下标志启用它们：

    -Dakka.test.tags.include=long-running
    -Dakka.test.tags.include=timing

或者如果你需要启用两者：

    -Dakka.test.tags.include=long-running,timing

##### 发布到Ivy本地存储库
如果你想要将项目部署到Ivy本地资源库 （例如，从 sbt 项目中使用）， 使用``publish-local``命令：

    sbt publish-local

> 注意

> Akka使用 ScalaDoc 为 API 文档生成类图。这需要安装 Graphviz 软件包的``dot``命令来避免错误。你可以通过添加``-Dakka.scaladoc.diagrams=false``标志来禁用图生成

##### sbt 交互模式
请注意在上面的例子我们调用 ``sbt compile`` 和 ``sbt test``等等，但sbt也有一种互动的模式。如果你键入``sbt``就进入了 sbt 交互式提示符，并可以直接输入命令。这节省了为每个命令启动一个新的 JVM 实例的开销，从而可以更快、更方便。

例如，Akka构建一般是这样做的：

       % sbt
       [info] Set current project to default (in build file:/.../akka/project/plugins/)
       [info] Set current project to akka (in build file:/.../akka/)
       > compile
       ...
       > test
       ...

##### sbt 批处理模式
也可以在单个调用中组合命令。例如，可以像这样测试和发布Akka到本地的Ivy资源库中：

    sbt test publish-local

### 依赖
你可以查看通过``sbt update``创建的Ivy依赖解析信息，并在 ``~/.ivy2/cache``中找到。例如，``~/.ivy2/cache/com.typesafe.akka-akka-remote-compile.xml`` 文件包含Akka远程模块编译依赖的解析信息。如果在 web 浏览器中打开此文件，你会得到一个易于导航的依赖关系视图。


