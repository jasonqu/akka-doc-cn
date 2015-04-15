# Agents

> 注：本节未经校验，如有问题欢迎提issue

Akka中的Agent是受 [Clojure agent](http://clojure.org/agents)启发的.

Agent 提供对独立位置的异步修改. Agent在其生命周期中绑定到一个单独的存储位置，对这个存储位置的数据的修改（到一个新的状态）仅允许作为一个操作的结果发生。 对其进行修改的操作是函数，该函数被异步地应用于Agent的状态，其返回值成为Agent的新状态。 Agent的状态应该是不可变的。

虽然对Agent的修改是异步的，但是其状态总是可以随时被任何线程 来获得(通过 ``get`` 或 ``apply``)而不需要发送消息。

Agent是响应式的（reactive）. 对所有agent的更新操作在一个``ExecutionContext``的不同线程中并发执行。在每一个时刻，每一个Agent最多只有一个 ``send`` 被执行. 从某个线程派发到agent上的操作的执行次序与其发送的次序一致，但有可能与从其它线程源派发来的操作交织在一起。

> 注意

> Agent对创建它们的节点是本地的。这意味着你一般不应包括它们在消息中，因为可能会被传递到远程actor或作为远程actor的构造函数参数；那些远程Actor不能读取或更新Agent。

###创建Agent
创建Agent时，调用 ``Agent(value)`` ，传入它的初始值并提供一个隐式的``ExecutionContext``供其使用，在这些例子中我们将使用默认的全局量，不过你的方法可能不同(YMMV : Your Method May Vary)：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import akka.agent.Agent
val agent = Agent(5)
```

###读取 Agent 的值
Agent可以用括号调用来去引用 (你可以获取一个Agent的值) ，像这样：

```scala
val result = agent()
```

或者使用 get 方法：

```scala
val result = agent.get
```

读取Agent的当前值不包括任何消息传递，并立即执行。所以说虽然Agent的更新的异步的，对它的状态的读取却是同步的。

###更新 Agent (send & alter)
更新Agent有两种方法：send一个函数来转换当前的值，或直接send一个新值。Agent会自动异步地应用新的值或函数。更新是以一种“发射后不管”的方式完成的，唯一的保证是它会被应用。 至于什么时候应用则没有保证，但是从同一个线程发到Agent的操作将被顺序应用。你通过调用``send``函数来应用一个值或函数。

```scala
// send a value, enqueues this change
// of the value of the Agent
agent send 7

// send a function, enqueues this change
// to the value of the Agent
agent send (_ + 1)
agent send (_ * 2)
```

你也可以在一个独立的线程中派发一个函数来改变其内部状态。这样将不使用响应式线程池，并可以被用于长时间运行或阻塞的操作。 相应的方法是 ``sendOff`` 。 派发器不管使用 ``sendOff`` 还是 ``send`` 都会顺序执行。

```scala
// the ExecutionContext you want to run the function on
implicit val ec = someExecutionContext()
// sendOff a function
agent sendOff longRunningOrBlockingFunction
```

所有的``send``都有一个对应的``alter``方法来返回一个``Future``。参考[Futures](01_futures.md)来获取``Future``的更多信息。

```scala
// alter a value
val f1: Future[Int] = agent alter 7

// alter a function
val f2: Future[Int] = agent alter (_ + 1)
val f3: Future[Int] = agent alter (_ * 2)
```

```scala
// the ExecutionContext you want to run the function on
implicit val ec = someExecutionContext()
// alterOff a function
val f4: Future[Int] = agent alterOff longRunningOrBlockingFunction
```

###等待Agent的返回值
也可以获得一个Agent值的``Future``，将在所有当前排队的更新请求都完成以后完成:

```scala
val future = agent.future
```

参考[Futures](01_futures.md)来获取``Future``的更多信息。

###Monadic 用法
Agent 也支持 monadic 操作, 这样你就可以用for-comprehensions对操作进行组合. 在 monadic 用法中, 旧的Agent不会变化，而是创建新的Agent。 所以老的值（Agents）仍像原来一样可用。这就是所谓的‘持久’.

monadic 用法示例:

```scala
import scala.concurrent.ExecutionContext.Implicits.global
val agent1 = Agent(3)
val agent2 = Agent(5)

// uses foreach
for (value <- agent1)
  println(value)

// uses map
val agent3 = for (value <- agent1) yield value + 1

// or using map directly
val agent4 = agent1 map (_ + 1)

// uses flatMap
val agent5 = for {
  value1 <- agent1
  value2 <- agent2
} yield value1 + value2
```

###配置
有一些配置属性是针对Agent模块的，请参阅[参考配置](../chapter2/09_configuration.md#config-akka-agent)。

###废弃的事务性Agent
Agent参与封闭 STM 事务是 2.3 废弃的功能。

如果Agent在一个封闭的事务中使用，然后它将参与该事务。如果你在一个事务内发送到Agent，然后对该Agent的派发将暂停直到该事务被提交，如果事务中止则丢弃该派发。下面是一个示例：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import akka.agent.Agent
import scala.concurrent.duration._
import scala.concurrent.stm._

def transfer(from: Agent[Int], to: Agent[Int], amount: Int): Boolean = {
  atomic { txn =>
    if (from.get < amount) false
    else {
      from send (_ - amount)
      to send (_ + amount)
      true
    }
  }
}

val from = Agent(100)
val to = Agent(20)
val ok = transfer(from, to, 50)

val fromValue = from.future // -> 50
val toValue = to.future // -> 70
```



