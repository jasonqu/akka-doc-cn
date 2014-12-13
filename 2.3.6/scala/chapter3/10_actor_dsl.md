# Actor DSL

###The Actor DSL
简单的actor——例如一次性worker甚或至在REPL中尝试的事物——可以更简洁地使用`Act`特质创建。支持的基础设施通过以下导入绑定：

```scala
import akka.actor.ActorDSL._
import akka.actor.ActorSystem
 
implicit val system = ActorSystem("demo")
```

在这一节的所有代码示例都假定有此导入。下面的所有示例中隐式actor系统都作为`ActorRefFactory`提供服务。要定义一个简单的actor，以下代码就足够了：

```scala
val a = actor(new Act {
  become {
    case "hello" ⇒ sender() ! "hi"
  }
})
```

在这里，`actor`方法，根据其调用的上下文，取代了``system.actorOf``或
``context.actorOf``：它需要一个隐式的 `ActorRefFactory`，其中在actor内可以通过以下方式获取``implicit val context: ActorContext``。在actor外，你不得不要声明隐式的`ActorSystem`，或者你可以显式地提供工厂（具体参见下文）。

两种发起``context.become``（更换或添加新的行为）的可能方式是分别提供的，从而支持不凌乱的嵌套接收标记语法：

```scala
val a = actor(new Act {
  become { // this will replace the initial (empty) behavior
    case "info" ⇒ sender() ! "A"
    case "switch" ⇒
      becomeStacked { // this will stack upon the "A" behavior
        case "info"   ⇒ sender() ! "B"
        case "switch" ⇒ unbecome() // return to the "A" behavior
      }
    case "lobotomize" ⇒ unbecome() // OH NOES: Actor.emptyBehavior
  }
})
```

请注意，调用 ``unbecome`` 比 ``becomeStacked`` 次数多将导致原始行为被安装，对`Act`特质来说是空行为（外部``become``只是在构造过程中替换它）。

#####生命周期管理

生命周期挂钩也可以暴露为 DSL 元素使用 （见[启动Hook](01_actors.md#start-hook-scala)和[终止Hook](01_actors.md#stop-hook-scala)），在那里如下所示的调用方法可以取代各自挂钩的内容：

```scala
val a = actor(new Act {
  whenStarting { testActor ! "started" }
  whenStopping { testActor ! "stopped" }
})
```

如果actor的逻辑生命周期匹配重新启动周期（即 ``whenStopping`` 在重新启动之前执行，并且`` whenStarting``在重启之后执行），上面的代码就足够了。如果这不是所期望的，可以使用下面的两个挂钩（请参阅[重启Hook](01_actors.md#restart-hook-scala)）：

```scala
val a = actor(new Act {
  become {
    case "die" ⇒ throw new Exception
  }
  whenFailing { case m @ (cause, msg) ⇒ testActor ! m }
  whenRestarted { cause ⇒ testActor ! cause }
})
```

另外还可以创建嵌套actors，即孙子actor，像这样：

```scala
// here we pass in the ActorRefFactory explicitly as an example
val a = actor(system, "fred")(new Act {
  val b = actor("barney")(new Act {
    whenStarting { context.parent ! ("hello from " + self.path) }
  })
  become {
    case x ⇒ testActor ! x
  }
})
```

> 注意

> 在某些情况下必须显式传递`ActorRefFactory`给`actor()`方法（当编译器告诉你出现了模糊蕴涵(implicits)时，你会发现的）。

孙子actor会被子actor监管；此外这类关系的监管策略也可以使用 DSL 元素进行配置（监管指令是`Act`特质的一部分）：

```scala
superviseWith(OneForOneStrategy() {
  case e: Exception if e.getMessage == "hello" ⇒ Stop
  case _: Exception                            ⇒ Resume
})
```

#####有`Stash`的actor

最后并且很重要的一点是：还有一点像魔法一样方便的内置，可以检测静态给定的actor子类型的运行时类，是不是通过`Stash`特质扩展 `RequiresMessageQueue `特质（这是一个复杂表述，即``new Act with Stash``不能工作，因为它的运行时类型被擦除为``Act``的匿名子类型）。目的是为自动根据`Stash`的需要使用适当的基于deque的邮箱类型。如果你想要使用这个魔法，只需扩展 `ActWithStash`：

```scala
val a = actor(new ActWithStash {
  become {
    case 1 ⇒ stash()
    case 2 ⇒
      testActor ! 2; unstashAll(); becomeStacked {
        case 1 ⇒ testActor ! 1; unbecome()
      }
  }
})
```

