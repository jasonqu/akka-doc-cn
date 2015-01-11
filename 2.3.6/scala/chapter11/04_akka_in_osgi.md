# Akka与OSGi


Configuring the OSGi Framework
------------------------------

To use Akka in an OSGi environment, the ``org.osgi.framework.bootdelegation``
property must be set to always delegate the ``sun.misc`` package to the boot classloader
instead of resolving it through the normal OSGi class space.

Activator
---------

To bootstrap Akka inside an OSGi environment, you can use the ``akka.osgi.ActorSystemActivator`` class
to conveniently set up the ActorSystem.

```scala
import akka.actor.{ Props, ActorSystem }
import org.osgi.framework.BundleContext
import akka.osgi.ActorSystemActivator

class Activator extends ActorSystemActivator {

  def configure(context: BundleContext, system: ActorSystem) {
    // optionally register the ActorSystem in the OSGi Service Registry
    registerService(context, system)

    val someActor = system.actorOf(Props[SomeActor], name = "someName")
    someActor ! SomeMessage
  }

}
```

The ``ActorSystemActivator`` creates the actor system with a class loader that finds resources
(``reference.conf`` files) and classes from the application bundle and all transitive dependencies.

The ``ActorSystemActivator`` class is included in the ``akka-osgi`` artifact::

      <dependency>
        <groupId>com.typesafe.akka</groupId>
        <artifactId>akka-osgi_@binVersion@</artifactId>
        <version>@version@</version>
      </dependency>


Sample
------

A complete sample project is provided in [akka-sample-osgi-dining-hakkers](http://github.com/akka/akka/tree/v2.3.6/akka-samples/akka-sample-osgi-dining-hakkers).
