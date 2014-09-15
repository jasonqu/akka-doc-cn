# 概述

* [术语，概念](01_What_is_Akka.md)
  * 并发 vs. 并行
  * 异步 vs. 同步
  * 非阻塞 vs. 阻塞
  * 死锁 vs. 饥饿 vs. 活锁
  * 竟态条件
  * 非阻塞担保（进展条件）
  * 推荐文献




Terminology, Concepts
Concurrency vs. Parallelism
Asynchronous vs. Synchronous
Non-blocking vs. Blocking
Deadlock vs. Starvation vs. Live-lock
Race Condition
Non-blocking Guarantees (Progress Conditions)
Recommended literature
Actor Systems
Hierarchical Structure
Configuration Container
Actor Best Practices
Blocking Needs Careful Management
What you should not concern yourself with
What is an Actor?
Actor Reference
State
Behavior
Mailbox
Children
Supervisor Strategy
When an Actor Terminates
Supervision and Monitoring
What Supervision Means
The Top-Level Supervisors
What Restarting Means
What Lifecycle Monitoring Means
One-For-One Strategy vs. All-For-One Strategy
Actor References, Paths and Addresses
What is an Actor Reference?
What is an Actor Path?
How are Actor References obtained?
Actor Reference and Path Equality
Reusing Actor Paths
The Interplay with Remote Deployment
What is the Address part used for?
Top-Level Scopes for Actor Paths
Location Transparency
Distributed by Default
Ways in which Transparency is Broken
How is Remoting Used?
Peer-to-Peer vs. Client-Server
Marking Points for Scaling Up with Routers
Akka and the Java Memory Model
The Java Memory Model
Actors and the Java Memory Model
Futures and the Java Memory Model
STM and the Java Memory Model
Actors and shared mutable state
Message Delivery Reliability
The General Rules
The Rules for In-JVM (Local) Message Sends
Higher-level abstractions
Dead Letters
Configuration
Where configuration is read from
When using JarJar, OneJar, Assembly or any jar-bundler
Custom application.conf
Including files
Logging of Configuration
A Word About ClassLoaders
Application specific settings
Configuring multiple ActorSystem
Reading configuration from a custom location
Actor Deployment Configuration
Listing of the Reference Configuration