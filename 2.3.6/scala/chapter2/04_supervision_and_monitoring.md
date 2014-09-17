# 监管与监控

这一节将简述监管背后的概念、原语及语义。要了解这些是如何转换成真实代码，请参阅相关的Scala和Java API章节。

###监管的意思
在 [Actor 系统]() 中说过，监管描述的是actor之间的依赖关系：监管者将任务委托给下属，并相应地对下属的失败状况进行响应。当一个下属出现了失败（即抛出一个异常），它自己会将自己和自己所有的下属挂起，然后向自己的监管者发送一个提示失败的消息。基于所监管的工作的性质和失败的性质，监管者可以有4种基本选择：

1 恢复下属，保持下属当前积累的内部状态
1 重启下属，清除下属的内部状态
1 永久地停止下属
1 升级失败（沿监管树向上传递失败），由此失败自己

始终要把一个actor视为整个监管树形体系的一部分是很重要的，这解释了第4种选择存在的意义（因为一个监管者同时也是其上方监管者的下属），并且隐含在前3种选择中：恢复actor会恢复其所有下属，重启一个actor也必须重启其所有下属（不过需要看下面的详述获取更多细节），类似地终止一个actor会终止其所有下属。需要强调[Actor]() 类的 [preRestart]() 钩子（hook）缺省行为是在重启前终止它的所有下属，但这个钩子可以被重写；对所有子actor的递归重启操作在这个钩子之后执行。

每个监管者都配置了一个函数，它将所有可能的失败原因（即异常）翻译成以上四种选择之一；注意，这个函数并不将失败actor的标识作为输入。我们很快会发现在有些结构中这种方式可能看起来不够灵活，例如会希望对不同的下属应用不同的策略。在这一点上我们一定要理解监管是为了组建一个递归的失败处理结构。如果你试图在某一个层次做太多事情，这个层次会变得复杂并难以理解，因此这时我们推荐的方法是增加一个监管层次。

Akka实现的是一种叫“父监管”的形式。Actor只能被其它的actor创建——顶部的actor由库来提供——每一个被创建的actor都由其父亲所监管。这种限制使得actor的监管结构隐式符合其树形层次，并提倡合理的设计方法。需要强调的是这也保证了actor不会成为孤儿或者拥有在系统外界的监管者（被外界意外捕获）。另外，这形成了对actor应用(或其子树)一种自然又干净的关闭过程。

Warning
警告

监管相关的父-子沟通，使用了特殊的系统消息及其固有的邮箱，从而和用户消息隔离开来。这意味着，监管相关的事件相对于普通的消息没有确定的顺序关系。在一般情况下，用户不能影响正常消息和失败通知的顺序。相关详细信息和示例，请参见讨论：[消息排序](message-ordering)。

###顶级监管者

.. image:: guardians.png
   :align: center
   :width: 360

一个actor系统在其创建过程中至少要启动三个actor，如上图所示。有关actor路径及相关信息请参见[Top-Level Scopes for Actor Paths.](toplevel-paths)。

#####`/user`: 守护Actor

这个名为`"/user"`的守护者，作为所有用户创建actor的父actor，可能是需要打交道最多的。使用`system.actorOf()`创建的actor都是其子actor。这意味着，当该守护者终止时，系统中所有的普通actor都将被关闭。同时也意味着，该守护者的监管策略决定了普通顶级actor是如何被监督的。自Akka 2.1起就可以使用这个设定`akka.actor.guardian-supervisor-strategy`，以一个`SupervisorStrategyConfigurator`的完整类名进行配置。当这个守护者上升一个失败，根守护者的响应是终止该守护者，从而关闭整个actor系统。

#####`/system`: 系统守护者

这个特殊的守护者被引入，是为了实现正确的关闭顺序，即日志（logging）要保持可用直到所有普通actor终止，即使日志本身也是用actor实现的。其实现方法是：系统守护者观察user守护者，并且观看该用户的监护人，并在收到`Terminated`消息初始化其自己的关闭过程。



并开始在接收到终止消息的其自己的关断来实现。顶级系统参与者使用的是这将无限期地重新启动后，所有类型的异常，除了ActorInitializationException和ActorKilledException，这将终止在孩子问题上的战略监督。所有其他可抛出都升级，这将关闭整个系统的演员。












重启的意思
当actor在处理消息时出现失败，失败的原因分成以上三类:

对收到的特定消息的系统错误（i.e. 程序错误）
处理消息时一些外部资源的（临时性）失败
actor内部状态崩溃了
Unless the failure is specifically recognizable, the third cause cannot be ruled out, which leads to the conclusion that the internal state needs to be cleared out. If the supervisor decides that its other children or itself is not affected by the corruption—e.g. because of conscious application of the error kernel pattern—it is therefore best to restart the child. This is carried out by creating a new instance of the underlying Actor class and replacing the failed instance with the fresh one inside the child’s ActorRef; the ability to do this is one of the reasons for encapsulating actors within special references. The new actor then resumes processing its mailbox, meaning that the restart is not visible outside of the actor itself with the notable exception that the message during which the failure occurred is not re-processed.

以下是重启过程中发生的事件的精确次序：

actor被挂起
调用旧实例的 supervisionStrategy.handleSupervisorFailing 方法 (缺省实现为挂起所有的子actor)
调用旧实例的 preRestart hook (缺省实现为向所有的子actor发送终止请求并调用 postStop)
等待所有子actor终止直到 preRestart 最终结束
调用旧实例的 supervisionStrategy.handleSupervisorRestarted 方法 (缺省实现为向所有剩下的子actor发送重启请求)
再次调用之前提供的actor工厂创建新的actor实例
对新实例调用 postRestart
恢复运行新的actor




