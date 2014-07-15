---
title: Akka DistributedPubSubExtension
date: 2014-07-16 23:30
author: Eric Weise
template: article.jade
---

Akka provides power clustering capabilities but reading through the <a href='http://doc.akka.io/docs/akka/2.3.2/scala/cluster-usage.html'>online docs</a> it may not be obvious how to set it up for your particular needs. For my use case, I wanted to create multiple actors representing a particular service, that would run on more that one machine. If one machine failed, actors on other machines could still handle the requests, as in a typical high availability scenario. In addition, I did not want the calling actor to have any knowledge of which machines could handle the request, only that one actor in the cluster would handle it. Akka's provides exactly these capabilities using the DistributedPubSubExtension provided in Akka's contrib module. This article will demonstrate how to create a clustered service using the DistributedPubSubExtension.
<p>In our example we have two types of actors; a BackendService actor which represents some arbitrary service running in the cluster and a WebService actor which handles http requests and in turn calls a BackendService to perform some work. The BackendService is stateless and there are many running in the cluster. The WebService just need to call any one BackendService to have the work performed.



## The DistributedPubSubMediator
Both the WebService and the BackEndService create a DistributedPubSubMediator. The DistributedPubSubMediator is an actor that maintains a registry of other ActorRefs and distributes them to peers around the network. This allows clients to refer actors by role instead of by a specific address. It also allows actors to join or leave the cluster and the mediator will track which actors are currently active.

```scala
  val mediator = DistributedPubSubExtension(context.system).mediator
```


## Backend Service Implementation

In our BackendServiceActor constructor we simply need to create the mediator and register our actor with it. The mediator will in turn update all its peer mediators in the cluster informing them that our BackendServiceActor has joined the cluster

```scala
  mediator ! Put(self)
```

The rest of the BackendActor is a standard Actor implementation where we listen for messages in a receive method

```scala
  def receive = {
    case PerformWork =>
      log.info("Backend Service is performing some work")
      sender() ! OK
  }
```

## The Web Service

For our Web service we will use Spray.io which is a library built on top of Akka that provides all the functionality we need to process http requests. To use Spray we create an actor that inherits from the HttpService trait.
```scala
   class WebServiceActor extends Actor with HttpService
```

Like in the BackendService actor, we need to create a mediator in the HttpService constructor. Then we can send messages to the BackendService via the mediator
```scala
mediator ? Send("/user/backend-service", PerformWork, false)
```

As with all actors we need to implement a 'receive' method. Our receive method will provide http routing information to Spray so it knows how to handle requests. Spray provides a DSL for doing this. In our example we implement just one route "dowork" in order to see our cluster in action

```scala
def receive = runRoute {
    path("dowork") {
      onComplete(mediator ? Send("/user/backend-service", PerformWork, false)) {
        case Success(value) => complete("OK")
        case Failure(e) => complete(e.getMessage)
      }
    }
  }
```

##Configuration
Both services need to provide configuration information to the ActorSystems. In the application.conf file we need to add the extension class

```
  extensions = [
    "akka.contrib.pattern.DistributedPubSubExtension"
  ]
```

You can also configure the PubSubExtension with the following properties
```
akka.contrib.cluster.pub-sub {
  # Actor name of the mediator actor, /user/distributedPubSubMediator
  name = distributedPubSubMediator

  # Start the mediator on members tagged with this role.
  # All members are used if undefined or empty.
  role = ""

  # How often the DistributedPubSubMediator should send out gossip information
  gossip-interval = 1s

  # Removed entries are pruned after this duration
  removed-time-to-live = 120s
}
```

##Conclusion
A complete example that can run is available  <a href="https://github.com/eweise/akka-pubsub-cluster-example">here</a>. Try booting up multiple BackendService instances. You will notice that the router selects actors randomly which is the default but can be changed to other strategies including round-robin. One issue I see in the example is that a delay occurs handling requests when a BackendService leaves the cluster, potentially causing the WebService requests to timeout for a brief time. This could possibly be hardened by adding retry logic into the Webservice. Overall Akka provides an incredible amount of functionality and the contrib module builds on top of the core features to making implementing clustered services almost trivial.

