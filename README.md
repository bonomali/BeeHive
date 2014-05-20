![BeeHive](https://raw.githubusercontent.com/aliostad/PublicImages/master/diagrams/BeeHive-164px.png)

BeeHive
=======

An Actor Helper mini-Library for cloud - currently for Windows Azure only. Implementation is very simple - if you need a complex implementation of the Actor Model, you are probably doing it wrong.

This library helps implementing purely the business logic and the rest is taken care of.

Business process is broken down into a series/cascade/tree of events where each step only knows the event it is consuming and the event(s) it is publishing. Actors define the name of the queue they are supposed to read from (in the form of *QueueName* for Azure Service Bus Queues or *TopicName*-*SubscriptionName* for Azure Topics) and system will feed these processor actors via **Factory Actors**. 

Error handling and retry mechanism is under development.

### No-non-sense Microservices Actor Framework for the cloud

BeeHive makes it rediculously easy to build highly decoupled and evolvable event-based cloud systems.

For theoretical discussions, please refer to the [infoq article](http://www.infoq.com/articles/reactive-cloud-actors).
