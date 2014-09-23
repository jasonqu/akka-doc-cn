# 必修的“Hello World”

将著名的问候语——“Hello World”——打印在控制台中，这个艰巨任务的actor版在[Typesafe Activator](http://www.typesafe.com/platform/getstarted)中一个名为[“Akka Main in Scala”](http://www.typesafe.com/activator/template/akka-sample-main-scala)的教程中有介绍。

这个教程展示了一个通用启动类`akka.Main`，它只需要一个命令行参数：应用主actor的类名。这个main方法会创建actor需要的底层运行环境，启动主actor，并且做好在主actor结束的时候关闭整个系统的准备。

另外还有一个名为[Hello Akka!](http://www.typesafe.com/activator/template/hello-akka) 的 [Typesafe Activator](http://www.typesafe.com/platform/getstarted)教程也是关于这一问题的，不过它更加深入的描述了Akka的基础。
