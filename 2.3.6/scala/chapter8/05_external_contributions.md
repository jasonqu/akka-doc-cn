# 外部贡献

> 注：本节未经校验，如有问题欢迎提issue

该子项目用来作为外部开发人员贡献模块的地方，随着时间的推移它可能会，也可能不会进入正式支持的代码中。这种转变可以发生的条件包括：

* 必须有足够的兴趣把模块列入标准的发行版中，
* 必须积极维护模块
* 代码质量必须足够好，允许由Akka核心开发团队有效的维护

如果贡献被发现不足以"起飞"，它可能会再次在稍后的时间移除。

### 买者自负

在此子项目中的模块并不都得服从小版本之间的二进制兼容规则。次要版本中可能引入破坏 API 的更改并且没有通知，只是基于用户的反馈意见。一个模块可能未经事先通知而丢弃。Typesafe订阅并不包括对这些模块的支持。

### 当前模块的列表

* [可靠的代理模式](http://doc.akka.io/docs/akka/2.3.6/contrib/reliable-proxy.html)
  * 可靠代理简介
  * 它保证的到底是什么？
    * 连接到目标
  * 如何使用它
    * 配置
  * Actor契约
    * 它处理的消息
    * 它发送的消息
    * 它升级的异常
    * 它的参数

* [Actor消息节流](http://doc.akka.io/docs/akka/2.3.6/contrib/throttle.html)
  * 介绍
  * 如何使用它
  * 保障

* [Java 日志记录 (JUL)](http://doc.akka.io/docs/akka/2.3.6/contrib/jul.html)

* [使用显式确认的邮箱](http://doc.akka.io/docs/akka/2.3.6/contrib/peek-mailbox.html)[

* [群集单例](http://doc.akka.io/docs/akka/2.3.6/contrib/cluster-singleton.html)
  * 需要意识到的潜在问题
  * 一个例子

* [群集Sharding](http://doc.akka.io/docs/akka/2.3.6/contrib/cluster-sharding.html)
  * 一个Java示例
  * 一个Scala示例
  * 它是如何工作
  * 仅代理模式
  * 钝化
  * 配置

* [分布式发布订阅群集](http://doc.akka.io/docs/akka/2.3.6/contrib/distributed-pub-sub.html)
  * 一个Java示例
  * 一个Scala示例
  * DistributedPubSubExtension

* [群集客户机](http://doc.akka.io/docs/akka/2.3.6/contrib/cluster-client.html)
  * 一个例子
  * ClusterReceptionistExtension

* [聚合器模式](http://doc.akka.io/docs/akka/2.3.6/contrib/aggregator.html)
  * 介绍
  * 使用
  * 样本用例 - AccountBalanceRetriever
  * 样本用例 - 多个响应聚合和链接
  * 陷阱

### 利用这些贡献的建议方式
由于Akka团队不限制到此子项目的更新，即使在二进制兼容的切换版本中，而且有可能在没有警告的情况下删除模块，因此建议将源文件复制到你自己的代码基中，更改包名称。通过这种方式，你可以选择何时更新或哪些修补程序需要包括进来（以保持二进制兼容性，如果需要的话），而且更高版本的Akka不可能破坏你的应用程序。

### 建议的贡献格式

每个贡献应该是一个自包含的单元，包括一个源文件或一个专门使用的包，没有对此子项目中其他模块的依赖；不过它可以依赖于Akka发布。这将确保贡献可以单独移动到标准发布中。该模块应在 ``akka.contrib`` 的一个子包中。

每个模块必须伴随一个测试套件验证其提供的功能，可能辅之以集成和单元测试。测试应遵守[开发者指南]()，并放在``src/test/scala`` 或``src/test/java``目录中（被测试的模块需要正确匹配包名）。例如，如果该模块被称为 ``akka.contrib.pattern.ReliableProxy``，则其测试套件应命名为 ``akka.contrib.pattern.ReliableProxySpec``。

每个模块还必须有用[reStructured Text](http://sphinx.pocoo.org/rest.html)编写的恰当的文档。文档应该是一个单一的 ``<module>.rst`` 文件，在``akka-contrib/docs``目录中，包括从``index.rst``（此文件）的链接。















