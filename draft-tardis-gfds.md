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

Establishing frameworks and libraries for developing and managing this systems presents significant challenges due to the need to abstract the complexities of distributed architectures and the management of its intricacies, while still providing enough flexibility and control for developers. Developers often require low-level access to certain aspects of the system, such as network management, fault tolerance mechanisms, security guarantees, no name a few, albeit requiring a flexible and fluid programming model, specially for beginners. Frameworks must provide high-level abstractions that simplify common tasks—like managing communication between nodes or handling failures, without hiding critical details that could lead to performance bottlenecks or incorrect system behavior.

Network libraries such as {{lib2p}}, offer modular and flexible networking framework that provides the building blocks for creating peer-to-peer networks, however come with a steep learning curve and interoperability issues. Alternatively, simulators like {{PeerSim}} allow developers to quickly develop and test distributed systems, but fail to model realist execution environments.

In this work we propose a Generic Framework for Building Dynamic Distributed Systems (GFDS) which aims to offer a set of tools, abstractions, and best practices to allow developers to design, deploy, and manage distributed applications that are dynamic, resilient, scalable, and fault-tolerant.
The framework is composed of an execution model that details and controls the life-cycle of protocols (the base unit in the framework) and an architecture detailing a set of managers to handle the different components and their interactions, while providing common APIs for enabling inter-protocol communication between the different elements in the stack.

## Document Structure

This document describes a Generic Framework for Building Dynamic Distributed Systems and its structured as follows:

 - {{protocol}} describes the base unit of interaction within the framework, the protocol,
 - {{architecture}} describes the framework architecture, its different components and their interaction,
 - {{executionmodel}} details the execution model of the framework and the life cycle of protocols,
 - {{api}} describes the programming interfaces offered by the framework to the application level for inter-protocol communication,
 - {{examples}} provides real world scenarios and examples.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Anatomy {#protocol}

The main unit of interaction on the framework is the *protocol*.
Protocols embed the logic implemented by the developer and use the abstractions provided by the framework to interact with other protocols being executed locally and to handle communication with other nodes. A process may execute an arbitrary number of protocols concurrently at any given time, and protocols can communicate with each other to cooperate and delegate tasks. Moreover, as it is very common in distributed protocols to capture certain behaviors, protocols can execute actions periodically with the use of timers (e.g., execute a garbage collection function).

Each protocol is composed by four main concepts that dictate the anatomy and life cycle of a protocol:

  - **state** which describes the inner state of the protocol, containing the necessary data and data structures to ensure its correct behavior. The state should be initialized in a specialized *init* function and can be mutated by means of interaction with other local protocols, through periodic timers that alter the state of the protocol and finally, through communication with other nodes running the some protocol on different machines,
  - **timer handlers** are meant to execute periodic or configured tasks. When the timer expires, a handler is executed with a user defined logic. Additionally, protocols are able to cancel timers if they are no longer relevant (e.g., a timer setup by the lack of an acknowledgement can be canceled if the acknowledged arrived in the mean time),
  - **inter-protocol handlers** are in charge of managing communication between protocols running on the same machine. These handlers are divided into two categories: one-to-one *requests/replies* and one-to-many *notifications*.
  - **communication handlers** which manages incoming and outgoing communication events through different interfaces. This events can materialized as messages, streams or other generic data transfer methods.

Further details regarding the architecture and functionality of handlers are provided in {{architecture}}.

With most of the complexity abstracted by the framework, the developer is able to focus on the logic of the protocol, without having to worry about the low-level aspects associated with building large scale systems (e.g., dealing with faults in the network layer).

## Protocol Initialization

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
  setupPeriodicTimer(uponGarbageCollectTimer, 100s);
  ...

  registerNotificationHandler(NeighborNotification, uponNeighborNotification)
  ...

  registerMessageHandler(BroadcastMessage, uponBroadcastMessage)
  ...
~~~

## Handlers

### Timer-Handlers

### Inter-Protocol Handlers

### Communication Handlers

## Procedures

Beyond the information defined above, protocols may need to execute procedures. This procedures can range from simple calculations on specific parameters, or inflations on the local state.
Thus, protocols should be allowed to declare an arbitrary number of procedures that can then be invoked by the protocol among the different handlers defined in the *init* function.

The declaration of a procedure should proceed as follows:

~~~
procedureName(args):
  //Perform computations on args

  return result;
~~~

A possible usage of a procedure within a protocol:

~~~
calcIntersection (set1, set2):
  return set1 ^ set2;

uponSetMessage(SetMessage: msg):
  intersect = this.calcIntersection(this.set, msg.set)
  this.set= intersect
  sendReply(SetUpdateReply(this.set), destProto)
~~~

## Example

# Architecture {#architecture}
<!---
Diagram que segue o do Babel e adaptador às novas coisas
-->

## Event Manager {#eventmanager}
<!---
Falar sobre a fila e como é que os requests/replies e timers são tratados
-->

## Configuration Manager {#configmanager}
<!---
Falar sobre a auto configuração e adaptabilidade em run time
-->

## Channel Manager {#channelmanager}
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
