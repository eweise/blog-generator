---
title: Building Clustered Services Using Akka
date: 2014-06-21 23:30
author: Eric Weise
template: article.jade
---

If you are reading this article then you are probably already familiar with Akka and some of the capabilities it provides. But Akka is a fairly large framework and it is sometimes not clear how to set it up for your particular use case. In this example I will demonstrate how to set it up for a typically use case I have and which is probably fairly common. The scenario is when you have one actor that needs to call a second actor and the second actor need to be highly available. So let's say you have an actor that handles http requests and it in turn calls a shopping cart actor which is remote. The shopping cart actor is part of the company's e-commerce application and therefore it is on different production schedule than the http handling actor. The web application will typically be fronted by a load balancer so that we can have many web applications running. If we need to upgrade the web applications, we simply restart them one at a time so that there are always some running to handle the incoming traffic.
Load balancing is the key to keeping our system alive while updating our system so it would be nice if Akka could provide this for us and indeed it does. But reading through the documentation


## Using the DistributedPubSubMediator
The DistributedPubSubMediator is an actor that maintains a registry of other ActorRefs and distributes them to peers around the network. This allows clients to refer actors by role instead of by a specific address. It also allows actors to join or unjoin the cluster and the mediator will track which actors are currently active.
<p>In our example we have two types of actors; a BackendService actor which represents some arbitrary function happening in the cluster and a WebService actor which handles http requests and in turn calls a BackendService to perform some work. The BackendService is stateless and there are many running in the cluster. The WebService just need to call any one BackendService to have the work performed.

### Backend Service Implementation

In our BackendServiceActor constructor we need to create a DistributedPubSubExtension actor and get a reference to its actorRef called the mediator.

```scala
  val mediator = DistributedPubSubExtension(context.system).mediator
```

Then we register our BackendServiceActor with the mediator. The mediator will in turn update all its peer mediators in the cluster that our BackendServiceActor has joined the cluster

```scala
  mediator ! Put(self)
```

The BackendServiceActor will receive a SubscribeAck message once the mediator has added it to the mediator's registry. Once that happens, the BackendServiceActor is ready to start receiving messages. Here is what the receive method loosks like

```scala
     def receive = {
       case SubscribeAck(Subscribe("backend-service", None, `self`)) ⇒
         println("subscribed")
       case PerformWork =>
         log.info("Backend Service is performing some work")
         sender() ! OK
       case m@_ => log.warning(s"Backend Service received unknown message $m ")
```


## The Web Service

For our Web service we will use Spray.io which is a library built on top of Akka that provides all the functionality we need to process http requests. To use Spray we create an actor that inherits from the HttpService trait.
```scala
   class WebServiceActor extends Actor with HttpService
```

In the constructor of the WebService actor we are going to set up a pub sub actor extension that will provide the clustering and location transparency to the BackendService actors

```scala
  val actorRefFactory = context
  val mediator = DistributedPubSubExtension(context.system).mediator
```

Now to send a message to our backend, we simply call the mediator
```scala
mediator ? Send("/user/microservice", PerformWork, false)
```

Notice that

As with all actors we need to implement a 'receive' method. Our receive method will provide http routing information to Spray so it knows how to handle requests. Spray provides a DSL for doing this. In our example we implement just one route "dowork" in order to see our cluster in action

```scala
     def receive = runRoute {
       path("dowork") {
         onComplete(mediator ? Send("/user/microservice", PerformWork, false)) {
           case Success(value) => complete("OK")
           case Failure(e) => complete(e.getMessage)
         }
       }

```

We call the 'path' method passing in the route we are defining as well as the code that will be invoked when the route is called. Now the interesting part is
 ```scala
 mediator ? Send("/user/microservice", PerformWork, false)
 ```


You use Akka by starting some actors

```scala
val x = 0
def foo(s:String):Unit = {
	println("helloworld")
}
```

