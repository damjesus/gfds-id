---
title: "A Generic Framework for Building Dynamic Distributed Systems (GFDS)"
abbrev: "GFDS"
category: std

docname: draft-tardis-gfds-latest
submissiontype: IETF

number:
date: 29/01/2025
consensus: true
v: 0
keyword:
 - distributed systems
 - dynamic environments
 - swarms

author:
 -  name: Diogo Jesus
    organization: TaRDIS
    street: NOVA Laboratory for Computer Science and Informatics (NOVA LINCS)
    email: da.jesus@fct.unl.pt
 -  name: João Leitão
    organization: TaRDIS
    street: NOVA Laboratory for Computer Science and Informatics (NOVA LINCS)
    email: jc.leitao@fct.unl.pt

normative:
  BCP190:

informative:
  lib2p:
    -: lib2p
    target: https://libp2p.io
    title: lib2p
  PeerSim:
    -: PeerSim
    target: https://peersim.sourceforge.net
    title: PeerSim


--- abstract

Building  and managing highly dynamic and heterogenous distributed systems  can prove to be quite challenging due to the great complexity and scale of such environments.
This documents specifies a Generic Framework for Building Dynamic Distributed Systems (GFDS), which composes a reference architecture and execution model for developing and managing these systems while providing high level abstractions to users.

--- middle

# Overview

Building dynamic distributed systems is a complex and challenging task due to the inherent unpredictability and scale of such systems. These systems often consist of multiple nodes that may be located in different geographic regions, and they need to collaborate seamlessly to provide services or process data. The difficulty arises in managing issues like network latency, node failures, variable load distribution, or even node displacement in particular environments. These systems must remain highly available and responsive even when individual components experience failures, which requires robust fault tolerance and self-healing mechanisms.

Maintaining scalability and flexibility as the system evolves proves to be quite endeavouring as well. As demand grows, the system must be able to dynamically scale by adding or removing nodes without disrupting ongoing operations, which requires sophisticated orchestration, auto-reconfiguration and adaptability. Moreover, dealing with the complexities of data consistency, synchronization, and ensuring that all nodes have a coherent view of the system state introduces a level of complexity that can be difficult to manage.

Establishing frameworks and libraries for developing and managing this systems presents significant challenges due to the need to abstract the complexities of distributed architectures and the management of its intricacies, while still providing enough flexibility and control for developers. Developers often require low-level access to certain aspects of the system, such as network management, fault tolerance mechanisms, security guarantees, to name a few, albeit requiring a flexible and fluid programming model, specially for beginners. Frameworks must provide high-level abstractions that simplify common tasks—like managing communication between nodes or handling failures, without hiding critical details that could lead to performance bottlenecks or incorrect system behavior.

Network libraries such as {{lib2p}}, offer modular and flexible networking framework that provides the building blocks for creating peer-to-peer networks, however come with a steep learning curve and interoperability issues. Alternatively, simulators like {{PeerSim}} allow developers to quickly develop and test distributed systems, but fail to model realist execution environments.

In this work we propose a Generic Framework for Building Dynamic Distributed Systems (GFDS) which aims to offer a set of tools, abstractions, and best practices to allow developers to design, deploy, and manage distributed applications that are dynamic, resilient, scalable, and fault-tolerant.
The framework is composed of an execution model that details and controls the life-cycle of protocols (the base unit in the framework) and an architecture detailing a set of managers to handle the different components and their interactions, while providing common APIs for enabling inter-protocol communication between the different elements in the stack.

## Document Structure

This document describes a Generic Framework for Building Dynamic Distributed Systems and its structured as follows:

 - {{protocol}} describes the base unit of interaction within the framework, the protocol,
 - {{architecture}} describes the framework architecture, its different components and their interaction,
 - {{executionmodel}} details the execution model of the framework and the life cycle of protocols,
 - {{api}} describes the programming interfaces offered by the framework to handle interaction with the application level, inter-protocol communication and wireless communication,
 - {{examples}} provides real world scenarios and examples.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following abbreviations are used throughout the document:

- API: Application Programming Interface


# Protocol Anatomy {#protocol}

The main unit of interaction on the framework is the *protocol*.
Protocols embed the logic implemented by the developer and use the abstractions provided by the framework to interact with other protocols being executed locally and to handle communication with other nodes. A process may execute an arbitrary number of protocols concurrently at any given time, and protocols can communicate with each other to cooperate and delegate tasks. Moreover, as it is very common in distributed protocols to capture certain behaviors, protocols can execute actions periodically with the use of timers (e.g., execute a garbage collection function).

Each protocol is composed by four main concepts that dictate the anatomy and life cycle of a protocol:

  - **state** which describes the inner state of the protocol, containing the necessary data and data structures to ensure its correct behavior. The state should be initialized in a specialized *init* function and can be mutated by means of interaction with other local protocols, through periodic timers that alter the state of the protocol and finally, through communication with other nodes running the some protocol on different machines,
  - **timer handlers** are meant to execute periodic or configured tasks. When the timer expires, a handler is executed with a user defined logic. Additionally, protocols are able to cancel timers if they are no longer relevant (e.g., a timer setup by the lack of an acknowledgement can be canceled if the acknowledged arrived in the mean time),
  - **inter-protocol handlers** are in charge of managing communication between protocols running on the same machine. These handlers are divided into two categories: one-to-one *requests/replies* and one-to-many *notifications*.
  - **communication handlers** which manages incoming and outgoing communication events through different interfaces. This events can materialized as messages, streams or other generic data transfer methods.

With most of the complexity abstracted by the framework, the developer is able to focus on the logic of the protocol, without having to worry about the low-level aspects associated with building large scale systems (e.g., dealing with faults in the network layer).

## Protocol Initialization {#protocol-init}

As described previously, each protocol should implement a special *init* function. This function is meant to be executed
exactly one time during the life-cycle of the protocol, and has three main purposes:

 1. Initialize the protocol's state, namely, its control variables, local data structures, etc.,
 2. Choose the preferred communication interfaces,
 3. Register the different handlers (i.e., timer, inter-protocol and communication handlers).

 A typical protocol initialization would be structured as follows:

~~~
init(properties):
  // 1 - Setup initial state
  ...

  // 2 - Communication interface
  preferences = {TCP, BLE}
  registerCommunicationInterfaces(preferences)

  // 3 - Handlers
  registerRequestHandler(BroadcastRequest, uponBroadcastRequest)
  ...

  registerReplyHandler(DeliverReply, uponDeliverReply)
  ...

  registerTimerHandler(GarbageCollectTimer, uponGarbageCollectTimer)
  ...

  subscribeNotificationHandler(NeighborUpNotification, uponNeighborUpNotification)
  ...

  registerTransmissionHandler(BroadcastData, uponBroadcastData)
  ...
~~~

## Handlers {#protocol-handlers}

Handlers operate as callback functions. During the initialization of a protocol, the developer is in charge of registering the handlers associated with the protocol and their respective callbacks. This way, when a event is dispatched to the protocol by the framework (e.g., due to a request arriving from another protocol, a message in the network, etc.), the callback associated with the respective event is rightfully triggered.

In this section we will focus on how the handles work at the protocol level. Further details regarding the architecture and internal life-cycle management of handlers by the runtime are provided in {{architecture}}.

Each type of handler has specific information due to their different nature, but their registration mostly consists on specifying two fields: 1) a type encapsulating the incoming information arriving at the handler, and 2) the *callback function* to be triggered when an event arrives. This can be depicted as follows, by analyzing a transmission example:

~~~
//Types definition
def BroadcastData {
  byte [] data,
  int hopCount,
  Peer origin

  ...
}

init(properties):
  ...
  registerTransmissionHandler(BroadcastData, uponBroadcastData)
  ...

uponBroadcastData(BroadcastData: msg, Peer: sender):
  //Handle data
~~~

Hence, at the protocol level, the developer only has to worry about defining the correct types associated with each handler and callback functions to manage event arrivals.

### Timer-Handlers {#protocol-handlers-timer}

Timer-handlers allow to prepare tasks that should be executed periodically or after a certain amount of time as passed. This is specially helpful to model certain aspects of distributed systems, such as cases where some kind of verification as to be done periodically to ensure consistency or to guarantee time-bound in certain aspects of a protocol.

Thus, we propose the following API to interact with timers at the protocol level:

~~~
registerTimerHandler(TimerType: timerType, TimerHandler: function)

setupTimer(Timer: timer, long: timeout) -> long

setupPeriodicTimer(Timer: timer, long: first, long: period) -> long

cancelTimer(long: timerID)
~~~

- *registerTimerHandler*, as the name implies, should only be invoked at the protocol initialization and ensures that all the timers are correctly registered. The function receives the timer type (which encapsulates the information to be passed on as argument) and the callback function to be triggered.

- *setupTimer* setups a timer that will be triggered after a timeout. After the timeout has passed the timer will cease to exist. The function returns an unique timerID that can be used to cancel the timer if needed.

- *periodicSetupTimer* setups a timer that will be triggered periodically (as indicated by the *period* parameter). It is possible to specify a timeout until the first trigger with the *first* parameter. Akin to *setupTimer*, this function also returns an unique timerID.

- *cancelTimer* allows to cancel a timer by passing its unique timerID. This can be extremely helpful in cases where a periodic action is no longer needed, or, for example, if a timeout is no longer needed due to the arrival of data. Protocols should be able to setup and cancel timers during their lifecycle.

An example of this is depicted below, in a simple example with synchronous communication among nodes:

~~~
//Types definition
def GarbageCollectionTimer(){
  long interval
}

def AckTimer(){
  string msgID
}

//Omitting state for clarity

init(properties):
  ...
  registerTimerHandler(GarbageCollectionTimer, uponGarbageCollectTimer)
  timer = GarbageCollectionTimer(1m)
  setupPeriodicTimer(timer, 10s, 100s);

  registerTimerHandler(AckTimer, uponAckTimer)
  ...

uponBroadcastData(BroadcastData: data, Peer: sender, CommunicationInterface: interface):
  this.blocks.add(msg)
  setupTimer(AckTimer(msg.id), 10s)

//Executed every 100 seconds
uponGarbageCollectTimer(GarbageCollectTimer: timer):
  this.blocks.remove(msg -> Time.now - msg.timestamp > timer.interval)

//Executed 10s after setupTimer invocation
uponAckTimer(GarbageCollectTimer: timer):
  if (!this.acks.contains(timer.msgID))
    // run fault checks

~~~


### Inter-Protocol Handlers {#protocol-handlers-ipc}

Inter-protocol handlers govern the interaction among protocols being executed in the same machine. Usually, each machine in a distributed system is running a stack of protocols (i.e., protocols for membership, propagation, storage, etc.) and they interact with each other to achieve composability and describe more complex behaviors. A simple example of this can be seen in a propagation protocol (e.g., multicast) that makes use of the neighbors provided by a lower-level protocol in charge of membership on top of an overlay-network.

As mentioned previously, this types of handlers are divided into two categories: *request/reply* handlers that are meant for one-to-one interaction among protocols, and *notifications* with one-to-many semantics. The first is especially of use in scenarios where a protocol intends to request the execution of a certain task to another protocol (e.g., the storage protocol asking for the propagation of an object to the protocol in charge of propagation) and expects to receive a reply after said task is completed. On the other hand, the second enables protocols to subscribe to
notifications, so that they are notified when a node triggers a new notification (e.g., a message arrived in a pub/sub protocol and every protocol that is interested in it should be notified).

Regarding requests and their respective replies, we propose the following API:

~~~
registerRequestHandler(RequestType: requestType, RequestHandler: function)

sendRequest(Request: request, Protocol: destProtocol)


registerReplyHandler(ReplyType: replyType, ReplyHandler: function)

sendReply(Reply: reply, Protocol: destProtocol)
~~~

 - *registerRequestHandler* should only be invoked in the *init* function and ensures that all the requests are correctly registered. The function receives the request type (which encapsulates the information to be passed on as argument to the handler) and the callback function to be triggered.
 - *sendRequest* allows a protocol to send a request to another protocol. The function receives an instance of the request and the corresponding destination protocol.
- *registerReplyHandler* is akin to *registerRequestHandler*, but for replies.
- *sendReply* behaves similarly to *sendRequest*, thus allowing protocols to issue replies. Is to be noted that replies are usually issued as a response to requests issued by other protocols.


~~~
subscribeNotificationHandler(NotificationType: notificationType, NotificationHandler: function)

triggerNotification(Notification: notification)
~~~

- *subscribeNotificationHandler* should only be invoked in the *init* function and guarantees the correct subscription of the protocol to notifications. The function receives the notification type, and a callback to handle incoming notifications.
- *triggerNotification* triggers a notification to be propagated to all protocols subscribed to it. The function receives an instance of the notification as argument.

This semantic allows developers to fine grain the specification of their protocols by using a declarative language that allows them to focus their efforts on implementing the algorithm logic.

We provide a brief example how this, with simple broadcast application:

~~~
//Types definition
def BroadcastRequest(){
  byte [] data
}

def DeliverReply(){
  byte [] data
}

def NeighborUpNotification(){
  Peer: peer
}

//Protocol state
state = {
  neighbors: Set<Peer>,
  myself: Peer
}

init(properties):
  ...
  registerRequestHandler(PingRequest,uponPingRequest)
  subscribeNotification(NeighborUpNotification,uponNeighborUpNotification)
  ...

uponPingRequest(PingRequest: request, Protocol: sourceProto):
  msg = BroadcastData(request.msg, this.myself)
  this.neighbors.forEach(peer -> transmitData(msg, peer))
  sendReply(DeliverReply(request.msg), sourceProto)

uponNeighborUpNotification(NeighborUpNotification: notification):
  this.neighbors.add(notification.peer)
~~~

Note that in the previous example there was no registration of DeliverReply. This is due to the fact that only the protocols wishing to receive that information, namely the protocol that sent the request, are obliged to register such handlers.

### Communication Handlers {#protocol-handlers-communication}

Finally, while inter-protocol handlers are in charge of dealing with the interaction of protocols being executed in the same node, communication handlers manage data arriving from different nodes.

With a great deal of complexity being abstracted by the framework, at the protocol level we concern on providing a generic API to the developer which allows to send and receive information from other nodes without having to deal with the intricacies of managing such connections.
Namely, we extend our framework to regard different technologies, ranging from the typical network-based protocols like TCP or UDP, to short-range technologies such as Bluetooth Low Energy (BLE) (more details are provided in the following sections).

Thus, at the protocol level, we suggest the following abstractions.

~~~
registerCommunicationInterfaces(CommunicationInterface[]: preferences)
~~~

 - *registerCommunicationInterfaces* allows a developer to specify its preferences regarding communication interfaces, namely, the underline technologies being used to transmit data to other nodes. With this information, the underlying communication manager tries to transmit data following the designated preferences. For example, if a user passes *{TCP, BLE}* as is preferences, the framework attempts to propagate information through TCP, and if it isn't able, resorts to BLE. If a set of preferences is not passed onto the function, the framework shall fallback to the default settings.

~~~
registerTransmissionHandler(DataType: dataType, DataHandler: function)
~~~

- *registerTransmissionHandler* should only be invoked in the *init* function and ensures that all the transmissions arriving at the node are properly handled. The function receives the data type (which encapsulates the information to be passed on as argument to the handler) and the callback function to be triggered when data arrives. Is of note that these data types, since meant for being sent/received through the network and other transmission mediums, should implement the proper serializers and deserializes.

~~~
transmitData(Data: data, Peer: destination)

transmitData(CommunicationInterface: interface, Data: data, Properties: props)
~~~

- *transmitData* can be invoked in two distinct ways. A simple version that only takes the data as argument and its destination and transmit data with the preferences stated in the initializer. On the other hand, a more specialized version is available where the developer, beyond specifying the data which whishes to send, also passes the communication interface to be used and a set of optional properties (props). The rational behind having a properties parameter is to allow a more rich expressiveness with certain communication interfaces (e.g., MQTT) where additional information has to be passed (e.g., passing the topic associated with it).

This abstractions allow protocols to have some control on how data is transmitted to other nodes, while hiding the complexity of dealing with such. On the other hand, if a protocol is only concerned with guaranteeing that information flows in and out of its host node, it can make use of the more simple abstractions following the default configuration embedded into the framework.

The following example illustrates what was presented.

~~~
//Types definition
PayloadData {
  string payload
}

AckData {
  long timestamp
}

PublishRequest {
  string topic,
  string msg
}

init():
  //Omitting request and notification registration for simplicity

  preferences= {UDP, BLE}
  registerCommunicationInterfaces(preferences)

  registerTransmissionHandler(PayloadData, uponPayloadData)
  ...

uponPublishRequest(PublishRequest: request, Protocol: sourceProto):
  props = {characteristic: "topic"}
  payload = PayloadData(request.msg)
  transmitData(BLE, payload, props)

uponPayloadData(PayloadData: data, Peer: sender, CommunicationInterface: interface):
  ackData = AckData(Time.now)
  transmitData(ackData,sender)
~~~

The different abstractions related to interactions with multiple protocols of different nature (i.e., subscribing to a topic in MQTT, listening to a characteristic in BLE, etc.) are still under development.

## Procedures {#protocol-handlers-procedures}

Beyond the information defined above that specifies the use of the abstractions provided by the framework, protocols may also need to execute procedures. This procedures can range from simple calculations on specific parameters, or inflations on the local state.
Thus, protocols should be allowed to declare an arbitrary number of procedures, and be able to invoke them inside the different handlers defined in the *init* function.

The declaration of a procedure should be as follows:

~~~
procedureName(args):
  //Perform computations on args

  return result;
~~~

A possible usage of a procedure within a protocol:

~~~
calcIntersection (set1, set2):
  return set1 ^ set2;

uponSetData(SetData: data, Peer: origin):
  intersect = calcIntersection(this.set, msg.set)
  this.set = intersect
  sendReply(SetUpdateReply(this.set), destProto)
~~~

## Example

In this section we will provide a simple but complete example of a protocol definition using the constructions stated above. Some definitions are omitted for clarity and succinctness.

Types Definition:

~~~
def BroadcastRequest{
  byte [] data
}

def BroadcastNotification {
  byte [] data
}

def NeighborUpNotification {
  Peer neighbor
}

def BroadcastData {
  byte [] data
  long timestamp
}

def GarbageCollectionTimer {
  long interval
}
~~~

Protocol:

~~~
state = {
  dataSet : Set<BroadcastData>,
  neighbors: Set<Peer>,
  protocolApp: Protocol
}

init(properties):
  this.dataSet = Set<BroadcastData>()
  this.neighbors = Set<Peer>()
  this.app = properties.protocol

  preferences= {TCP, UDP}
  registerCommunicationInterfaces(preferences)

  registerRequestHandler(BroadcastRequest, uponBroadcastRequest)
  subscribeNotification(NeighborUpNotification,uponNeighborUpNotification)

  registerTimerHandler(GarbageCollectionTimer, uponGarbageCollectionTimer)
  setupPeriodicTimer(GarbageCollectionTimer(properties.interval), uponGarbageCollectionTimer)

  registerTransmissionHandler(BroadcastData, uponBroadcastData)

// Request/Reply Handlers
uponBroadcastRequest(BroadcastRequest: request, Protocol: sourceProtocol):
  broadcastData = BroadcastData(request.data, Time.now)
  deliver(broadcastData)
  propagate(this.neighbors, broadcastData)

// Timer Handlers
uponGarbageCollectionTimer(GarbageCollectionTimer: timer):
  this.dataSet.removeIf(data -> Time.now - data.timestamp > timer.interval)

// Notification Handlers
uponNeighborUpNotification(NeighborUpNotification: notification):
  this.neighbors.add(notification.neighbor)

// Communication Handlers
uponBroadcastData(BroadcastData:data, Peer: sender):
  deliver(data)
  propagate(this.neighbors - sender, data)

// Procedures
deliver(BroadcastData: broadcastData):
  if(!this.dataSet.contains(broadcastData)):
    this.dataSet.add(broadcastData)

    notification = BroadcastNotification(broadcastData.data)
    triggerNotification(notification)

propagate(Set<Peer> destinations, BroadcastData: broadcastData):
  destinations.forEach(n -> transmitData(broadcastData,n))
~~~

# Architecture {#architecture}
<!---
Diagram que segue o do Babel e adaptador às novas coisas
-->

         +---------------------------------------------------------+
         | +--------------+   +--------------+   +--------------+  |
         | |              |   |              |   |              |  |
         | |  Protocol 1  |   |  Protocol 2  |...|  Protocol N  |  |
         | |              |   |              |   |              |  |
         | +--------------+   +--------------+   +--------------+  |
         |  | ^                | ^                | ^              |
         |  | | +-------+      | | +-------+      | | +-------+    |
         |  | | | Event |      | | | Event |      | | | Event |    |
         |  | | | Queue |      | | | Queue |      | | | Queue |    |
         |  | | +-------+      | | +-------+      | | +-------+    |
         |  | |                | |                | |              |
         |  v |                v |                v |              |
         | +----------------------------------------------------+  |
         | |                                        +---------+ |  |
         | |                        Event           |  Timer  | |  |
         | |                       Manager          | Manager | |  |
         | |                                        +---------+ |  |
         | +----------------------------------------------------+  |
         |         ^                                    |          |
         |         |                                    v          |
         | +----------------------------------------------------+  |
         | |                                                    |  |
         | |                    Configuration                   |  |
         | |                       Manager                      |  |
         | |                                                    |  |
         | +----------------------------------------------------+  |
         |         ^                                    |          |
         |         |                                    v          |
         | +----------------------+       +---------------------+  |
         | |                      | ----> |                     |  |
         | |       Channel        |       |      Security       |  |
         | |       Manager        |       |       Manager       |  |
         | |                      | <---- |                     |  |
         | +----------------------+       +---------------------+  |
         +---------------------------------------------------------+
                        ^ ^                         ^ ^
                        | |                         | |
      <-----------------+ |                         | +--------------->
     <--------------------+                         +------------------>
                             Incoming and Outgoing
                                  Connections

## Event Manager {#eventmanager}
<!---
Falar sobre a fila e como é que os requests/replies e timers são tratados
-->

## Configuration Manager {#configmanager}
<!---
Falar sobre a auto configuração e adaptabilidade em run time
-->

## Channel Manager {#channelmanager}
Handling communication can prove to be extremely complex due to the different nature of different communication protocols.
<!---
Falar sobre o channel manager e a questao do multiplexer
-->

## Security Manager {#securitymanager}
<!---
Falar sobreo identity manager e essas coisas
-->

# Execution Model {#executionmodel}
<!---
Isto fala sobre o metodo de concorrencia (i.e, a fila), threading, etc. Ver o artigo do babel para inspiração
Colocar aqui aquela parte da main em que se coloca os protocols e se passa a configuraçao
-->

Each protocol is exclusively assigned a dedicated thread, which handles received events in a serial fashion by executing their respective callbacks. In a single Babel process, any number of protocols may be executing simultaneously, allowing multiple protocols to cooperate (i.e., multi-threaded execution), while shielding developers from concurrency issues, as all communication between protocols is done via message passing.
From the developer’s point-of-view, a protocol is responsible for defining the callbacks used to process the different types of events in its queue. The developer registers the callback for each type of event and implements its logic, while Babel handles the events by invoking their appropriate callbacks. While relatively simple, the event-oriented model provided by Babel allows the implementation of complex distributed protocols by allowing the developer to focus almost exclusively on the actual logic of the protocol, with minimal effort on setting up all the additional operational aspects.

# API {#api}

# Examples {#examples}

# Security Considerations

(Mandatory chapter)

# IANA Considerations

(Mandatory chapter)

# Implementation Status


--- back

# Acknowledgments
{:numbered="false"}

This work was partly funded by EU Horizon Europe under Grant Agreement no. 101093006 (TaRDIS) and FCT-Portugal under grant UIDB/04516/2020.
