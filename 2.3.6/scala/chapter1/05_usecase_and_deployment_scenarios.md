# 用例和部署场景

###我该如何使用和部署 Akka?

Akka 可以有几种使用方式:

* 作为一个库: 以普通jar包的形式放在classpath上，或放到web应用中的 `WEB-INF/lib`位置
* 作为一个独立的应用程序，使用[微内核(Scala)](../chapter6/07_microkernel.md) / [微内核(Java)](#TODO) ，自己使用一个main类来初始化Actor系统


#####将Akka作为一个库
当编写web应用的时候，你很可能要使用这种方式。通过添加更多的模块，可以有多种使用Akka库模式的方式。

#####将Akka用作单独的微内核
Akka 也可以作为独立微内核使用。参阅 [微内核(Scala)](../chapter6/07_microkernel.md) / [微内核(Java)](#TODO) 获取更多信息.

