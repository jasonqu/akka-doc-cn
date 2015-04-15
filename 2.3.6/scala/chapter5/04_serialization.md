# 序列化

> 注：本节未经校验，如有问题欢迎提issue

Akka 提供了内置的支持序列化的扩展, 你可以使用内置的序列化功能，也可以自己写一个.

内置的序列化功能被Akka内部用来序列化消息，你也可以用它做其它的序列化工作。

###用法
#####配置
为了让 Akka 知道对什么任务使用哪个``Serializer``, 你需要编辑你的 [配置文件](../chapter2/09_configuration.md), 在 “akka.actor.serializers”一节将名称绑定为``akka.serialization.Serializer``的实现，像这样:

```
akka {
  actor {
    serializers {
      java = "akka.serialization.JavaSerializer"
      proto = "akka.remote.serialization.ProtobufSerializer"
      myown = "docs.serialization.MyOwnSerializer"
    }
  }
}
```

在将名称与``Serializer``的不同实现绑定后，你需要指定哪些类的序列化使用哪种``Serializer``, 这部分配置写在“akka.actor.serialization-bindings”一节中:

```
akka {
  actor {
    serializers {
      java = "akka.serialization.JavaSerializer"
      proto = "akka.remote.serialization.ProtobufSerializer"
      myown = "docs.serialization.MyOwnSerializer"
    }

    serialization-bindings {
      "java.lang.String" = java
      "docs.serialization.Customer" = java
      "com.google.protobuf.Message" = proto
      "docs.serialization.MyOwnSerializable" = myown
      "java.lang.Boolean" = myown
    }
  }
}
```

你只需要指定消息的接口或抽象基类. 当消息实现了配置中多个类时，为避免歧义, 将使用最具体的类, 即对其它所有配置指定类，它都是子类的那一个. 如果这个条件不满足, 例如因为同时配置``java.io.Serializable``和``MyOwnSerializable``，而彼此都不是对方的子类型，将生成一个警告。

Akka 缺省提供使用 ``java.io.Serializable``和 [protobuf](http://code.google.com/p/protobuf/) ``com.google.protobuf.GeneratedMessage`` 的序列化工具 (后者仅当定义了对akka-remote模块的依赖时才有), 所以通常你不需要添加这两种配置; 由于 ``com.google.protobuf.GeneratedMessage`` 实现了 ``java.io.Serializable``, 在不特别指定的情况下，protobuf 消息将总是使用 protobuf 协议来做序列化. 要禁止缺省的序列化工具，将其对应的类型设为 “none”:

```
akka.actor.serialization-bindings {
  "java.io.Serializable" = none
}
```

###确认
如果你希望确认你的消息是可以被序列化的，你可以打开这个配置项:

```
akka {
  actor {
    serialize-messages = on
  }
}
```
> 警告

> 我们推荐只在运行测试代码的时候才打开这个选项. 在其它的场景打开它完全没有道理.

如果你希望确认你的 Props 可以被序列化, 你可以打开这个配置项:

```
akka {
  actor {
    serialize-creators = on
  }
}
```

> 警告

> 我们推荐只在运行测试代码的时候才打开这个选项. 在其它的场景打开它完全没有道理.

#####通过代码
如果你希望通过代码使用 Akka Serialization来进行序列化/反序列化, 以下是一些例子:

```scala
import akka.actor.{ ActorRef, ActorSystem }
import akka.serialization._
import com.typesafe.config.ConfigFactory

    val system = ActorSystem("example")

    // Get the Serialization Extension
    val serialization = SerializationExtension(system)

    // Have something to serialize
    val original = "woohoo"

    // Find the Serializer for it
    val serializer = serialization.findSerializerFor(original)

    // Turn it into bytes
    val bytes = serializer.toBinary(original)

    // Turn it back into an object
    val back = serializer.fromBinary(bytes, manifest = None)

    // Voilá!
    back should be(original)
```

更多信息请见 ``akka.serialization._`` 的 ``ScalaDoc``

###自定义
如果你希望创建自己的 ``Serializer``, 应该已经看到配置例子中的 ``docs.serialization.MyOwnSerializer`` 了吧?

#####创建新的 Serializer
首先你需要为你的 ``Serializer`` 写一个类定义，像这样:

```scala
import akka.actor.{ ActorRef, ActorSystem }
import akka.serialization._
import com.typesafe.config.ConfigFactory

class MyOwnSerializer extends Serializer {

  // This is whether "fromBinary" requires a "clazz" or not
  def includeManifest: Boolean = false

  // Pick a unique identifier for your Serializer,
  // you've got a couple of billions to choose from,
  // 0 - 16 is reserved by Akka itself
  def identifier = 1234567

  // "toBinary" serializes the given object to an Array of Bytes
  def toBinary(obj: AnyRef): Array[Byte] = {
    // Put the code that serializes the object here
    // ... ...
  }

  // "fromBinary" deserializes the given array,
  // using the type hint (if any, see "includeManifest" above)
  // into the optionally provided classLoader.
  def fromBinary(bytes: Array[Byte],
                 clazz: Option[Class[_]]): AnyRef = {
    // Put your code that deserializes here
    // ... ...
  }
}
```

然后你只需要做填空，在 [配置文件](../chapter2/09_configuration.md) 中将它绑定到一个名称， 然后列出需要用它来做序列化的类即可.

#####Actor引用的序列化
所有的 ActorRef 都是用 JavaSerializer进行序列化的, 但如果你写了自己的serializer， 你可能想知道如何正确对它们进行序列化和反序列化。在一般情况下，要使用的本地地址取决于作为序列化信息收件人的远程地址的类型。像这样使用``Serialization.serializedActorPath(actorRef)``：

```scala
import akka.actor.{ ActorRef, ActorSystem }
import akka.serialization._
import com.typesafe.config.ConfigFactory

    // Serialize
    // (beneath toBinary)
    val identifier: String = Serialization.serializedActorPath(theActorRef)

    // Then just serialize the identifier however you like

    // Deserialize
    // (beneath fromBinary)
    val deserializedActorRef = extendedSystem.provider.resolveActorRef(identifier)
    // Then just use the ActorRef
```

这是假定序列化发生在通过远程传输发送消息的上下文中。不过有一些序列化有其他用途，例如在actor应用程序之外存储actor引用（数据库等）。在这种情况下，需要牢记的重要一点是，actor路径的地址部分决定actor如何被通信连通。存储一个本地actor路径可能是正确的选择，如果回取发生在相同的逻辑上下文，但当在不同的网络主机上反序列化时它还不够： 为此，它将需要包括系统的远程传输地址。一个actor系统并不局限于只有一个远程运输，使得这个问题变得更有意思。当向``remoteAddr``发送消息时，要找出恰当的地址，你可以像下面这样使用`ActorRefProvider.getExternalAddressFor(remoteAddr)`：

```scala
object ExternalAddress extends ExtensionKey[ExternalAddressExt]

class ExternalAddressExt(system: ExtendedActorSystem) extends Extension {
  def addressFor(remoteAddr: Address): Address =
    system.provider.getExternalAddressFor(remoteAddr) getOrElse
      (throw new UnsupportedOperationException("cannot send to " + remoteAddr))
}

def serializeTo(ref: ActorRef, remote: Address): String =
  ref.path.toSerializationFormatWithAddress(ExternalAddress(extendedSystem).
    addressFor(remote))
```

> 注意

> 如果地址并没有``host``和``port``组件时``ActorPath.toSerializationFormatWithAddress``不同于``toString``，即它只为本地地址插入地址信息。

> ``toSerializationFormatWithAddress`` 还为actor添加了唯一 id，当actor终止，然后又按照相同名称重新创建时，该id将改变。将消息发送到指向那个老actor的引用，将不会传递给新的actor。如果你不想要这种行为，例如在长期存储引用的情况下，你可以使用不包括的唯一id的 ``toStringWithAddress``。

这就要求你至少知道哪种类型的地址将被反序列化生成的actor引用的系统支持；如果你手头没有具体的地址，你可以创建一个虚拟使用正确协议的地址 ``Address(protocol, "", "", 0)``（假定所用的实际传输与Akka的``RemoteActorRefProvider``是相兼容的）。

也有一个用来支持集群的默认远程地址（并且典型系统只有这一个）；你可以像这样获取它：

```scala
object ExternalAddress extends ExtensionKey[ExternalAddressExt]

class ExternalAddressExt(system: ExtendedActorSystem) extends Extension {
  def addressForAkka: Address = system.provider.getDefaultAddress
}

def serializeAkkaDefault(ref: ActorRef): String =
  ref.path.toSerializationFormatWithAddress(ExternalAddress(theActorSystem).
    addressForAkka)
```

#####Actor的深度序列化
做内部actor状态深度序列化，推荐的方法是使用[Akka持久化](../chapter3/08_persistence.md)。

###关于 Java 序列化
如果在做Java序列化时没有使用`JavaSerializer`, 你必须保证在动态变量``JavaSerializer.currentSystem``中提供一个有效的 ``ExtendedActorSystem`` . 它用在读取`ActorRef`表示时将其字符串表示转换成实际的引用. 动态变量``DynamicVariable`` 是一个 thread-local变量，所以在反序列化任何可能包含actor引用的数据时要保证这个变量有值.

###外部 Akka Serializers

[Akka-protostuff by Roman Levenstein](https://github.com/romix/akka-protostuff-serialization)


[Akka-quickser by Roman Levenstein](https://github.com/romix/akka-quickser-serialization)


[Akka-kryo by Roman Levenstein](https://github.com/romix/akka-kryo-serialization)


