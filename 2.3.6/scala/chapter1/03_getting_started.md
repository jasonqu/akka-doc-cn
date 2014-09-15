# 入门

###准备工作
Akka要求你安装了 [Java 1.6](http://www.oracle.com/technetwork/java/javase/downloads/index.html)或更高版本。

###入门指南和模板工程
组号的学习Akka的方法是下载[“Typesafe Activator”](http://www.typesafe.com/platform/getstarted?_ga=1.206427601.315817771.1410711243)并且尝试一下其中的Akka模板工程。

###下载
下载Akka有几种方法。你可以通过下载Typesafe平台来下载Akka（如前所述）。你可以下载包含微内核的完整发布包（包含所有的模块）. 或者也可以使用构建工具如Maven或SBT从Akka Maven仓库下载依赖。


###模块
Akka的模块化做得非常好，它为不同的功能提供了不同的Jar包。

* akka-actor – 标准Actor, 类型Actor，IO Actor等。
* akka-agent – Agent，与 Scala STM 集成
* akka-camel – Apache Camel 集成
* akka-cluster – 集群成员管理，弹性路由器。
* akka-kernel – Akka 微内核来运行简单应用服务器
* akka-osgi – 在OSGi容器中使用Akka的基本组件，包含akka-actor类
* akka-osgi-aries – Aries 的actor系统蓝图
* akka-remote.jar – 远程Actor
* akka-slf4j.jar – SLF4J日志(事件总线监听器)
* akka-testkit.jar – Actor系统的测试工具包
* akka-zeromq – ZeroMQ 集成

除了这些稳定的模块之外，还有一些虽然趋于稳定但仍然被标记为“实验”的模块。这并不是说他们的功能不符合预期，而主要的意思是他们的API还没有足够稳定到被认为已经固定了。你可以通过在我们的邮件组里进行反馈，来加速试验模块发布的进程。

* akka-contrib – 一系列的Akka贡献，他们有可能被加入核心模块中，详情见External Contributions http://doc.akka.io/docs/akka/2.3.6/contrib/index.html#akka-contrib。

实际的jar包文件名会加上版本号，如akka-actor_2.10-2.3.6.jar（对其他模块也是类似）。

查看Akka模块之间的jar依赖的详情在[依赖](http://doc.akka.io/docs/akka/2.3.6/dev/building-akka.html#dependencies)这一节中。


###使用发布版
从http://akka.io/downloads下载发布包并解压.

###使用快照版
Akka的每日快照发布在 http://repo.akka.io/snapshots/ 版本号中包含 SNAPSHOT 和时间戳. 你可以选择一个快照版，可以决定何时升级到一个新的版本。Akka快照仓库也可以在 http://repo.typesafe.com/typesafe/snapshots/找到，此处还包含Akka模块依赖的其它仓库。

<div class="admonition warning">
<p class="first admonition-title">Warning</p>
<p class="last">不鼓励直接使用Akka快照版（SNAPSHOT）、每日构建版（nightly）和里程碑版（milestone），除非你知道自己在做什么。</p>
</div>

###微内核
Akka发布包包含微内核。要运行微内核，将你应用的jar包放到 `deploy` 目录下并运行 `bin` 目录下的脚本即可。

关于微内核的更多文档在 [Microkernel (Scala)](http://www.gtan.com/akka_doc/modules/microkernel.html#microkernel) / Microkernel (Java)。

###使用构建工具

Akka可以与支持Maven仓库的构建工具一起使用。

###Maven仓库

对Akka 2.1-M2 及以后的版本：

[Maven Central](http://repo1.maven.org/maven2/)

对以前的Akka 版本：

[Akka Repo Typesafe Repo](http://repo.akka.io/releases/?_ga=1.215923159.1916581926.1392627894)

###通过Maven使用Akka

通过Maven使用Akka最简单的入门是检出[“Typesafe Activator”](http://www.typesafe.com/platform/getstarted?_ga=1.258920778.1916581926.1392627894)中的模板工程[“Akka Main in Java”](http://www.typesafe.com/activator/template/akka-sample-main-java?_ga=1.258920778.1916581926.1392627894)。

由于Akka已经发布到Maven中心仓库了（自2.1-M2版本起），所以直接在POM文件中加入Akka依赖即可。例如。这是akka-actor的依赖：

	<dependency>
	  <groupId>com.typesafe.akka</groupId>
	  <artifactId>akka-actor_2.10</artifactId>
	  <version>2.3.6</version>
	</dependency>


注意：对快照版本，SNAPSHOT和时间戳都在版本号中。

###通过SBT使用Akka

通过SBT使用Akka最简单的入门是检出[“Akka/SBT”](http://www.typesafe.com/resources/getting-started/typesafe-stack/downloading-installing.html?_ga=1.257948875.1916581926.1392627894#template-projects-for-scala-akka-and-play)模板工程。

通过SBT使用Akka的要点:

SBT安装指导 https://github.com/harrah/xsbt/wiki/Setup

`build.sbt` 文件:

	name := "My Project"
	 
	version := "1.0"
	 
	scalaVersion := "2.10.4"
	 
	resolvers += "Typesafe Repository" at "http://repo.typesafe.com/typesafe/releases/"
	 
	libraryDependencies +=
	  "com.typesafe.akka" %% "akka-actor" % "2.3.6"

注意：以上的`libraryDependencies`设置需要SBT 0.12.x或更高的版本。如果你使用更老版本的SBT，`libraryDependencies`需啊哟这样设置：

	libraryDependencies +=
	  "com.typesafe.akka" % "akka-actor_2.10" % "2.3.6"

















