---
title: AQS框架概述(译文)
date: 2022-04-04 10:16:55
tags:
- 并发
- 多线程
categories:
- code
- concurrence
---

## 摘要

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
Most synchronizers (locks, barriers, etc.) in the J2SE1.5 java.util.concurrent package 
are constructed using a small framework based on class AbstractQueuedSynchro- nizer.
This framework provides common mechanics for atomically managing synchronization state,
blocking and unblocking threads, and queuing. The paper describes the rationale, design, 
implementation, usage, and performance of this framework
</pre>
</details>

##### 译文

大部分的同步器(锁, 屏障, etc.) 在J2SE1.5 `java.util.concurrent` 并发集合包
是使用一个`AbstractQueuedSynchronizer`作为基类的轻量级框架构建的. 这个框架为
自动管理同步状态, 阻塞唤醒线程, 排队提供通用的机制. 这篇论文对这个框架的原理, 设计
, 实现, 用法和性能做了阐述.


## 1. 介绍

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h2>1. INTRODUCTION</h2>
Javatm release J2SE-1.5 introduces package java.util.concurrent, a collection of medium-level
concurrency support classes created via Java Community Process (JCP) Java Specification Request (JSR) 166.
Among these components are a set of synchronizers – abstract data type (ADT) classes that maintain an 
internal synchronization state (for example, representing whether a lock is locked or unlocked), 
operations to update and inspect that state, and at least one method that will cause a calling thread 
to block if the state requires it, resuming when some other thread changes the synchronization state to permit it.
Examples include various forms of mutual exclusion locks, read-write locks, semaphores, barriers, futures,
event indicators, and handoff queues.

As is well-known (see e.g., [2]) nearly any synchronizer can be used to implement nearly any other. 
For example, it is possible to build semaphores from reentrant locks, and vice versa. However, 
doing so often entails enough complexity, overhead, and inflexibility to be at best a second-rate engineering option.
Further, it is conceptually unattractive. If none of these constructs are intrinsically more primitive than the others,
developers should not be compelled to arbitrarily choose one of them as a basis for building others. Instead, 
JSR166 establishes a small framework centered on class AbstractQueuedSynchro- nizer, that provides common
mechanics that are used by most of the provided synchronizers in the package,
as well as other classes that users may define themselves.

The remainder of this paper discusses the requirements for this framework, the main ideas behind
its design and implementation, sample usages, and some measurements showing its performance characteristics.
</pre>
</details>

##### 译文

Java团队根据Java社区流程(JCP)Java特别请求(JSR)166发布J2SE-1.5时引入了`java.uti.concurrent`并发集合包, 
一个中间件基本的并发支持类. 这些组件是一个同步器的集合, 抽象数据类型(ADT)类维护一个内部的同步状态(例如, 代表一把锁是锁住还是未锁住),
操作来更新或者监控这个状态, 并且当状态需要的时候至少提供一个方法用来调用线程阻塞, 当其他线程改变这个这个同步状态时可以唤醒.
例子包括各种形式的互斥锁, 读写锁, 信号量, 屏障, 未来任务, 事件指示, 以及传递队列.

正如已知的几乎所有同步器都可以用来实现其他的同步器. 例如, 可以使用一个可重入锁可以构建信号量, 反之亦然.
然而, 这么做必定会带来大量的复杂度, 堆叠, 以及没有扩展性, 这是一个二流的工程选择.
进一步讲, 这么做概念上毫无吸引力. 如果这些结构本质上并没有比其他的更原生, 那么开发们不应该强行的选择它们中的一个来作为构建其他的基础.
作为代替, JSR166以 `AbstractQueuedSynchronizer` 构建了一个轻量框架, 为并发包里的同步器提供了通用的机制, 以及一些其他的用户
可以自定义的类.

本论文的主旨是讨论这个框架的需求, 设计和实现的主要思想, 简单的用法, 以及对它的性能指数的一些测量.


## 2. 需求

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h2>2. REQUIREMENTS</h2>
<h3>2.1 Functionality</h3>
Synchronizers possess two kinds of methods [7]: at least one acquire operation that blocks
the calling thread unless/until the synchronization state allows it to proceed, 
and at least one release operation that changes synchronization state in a way 
that may allow one or more blocked threads to unblock.
The java.util.concurrent package does not define a single unified API for synchronizers.
Some are defined via common interfaces (e.g., Lock), but others contain only specialized versions. 
So, acquire and release operations take a range of names and forms across different classes. 
For example, methods Lock.lock, Semaphore.acquire, CountDownLatch.await, and FutureTask.get 
all map to acquire operations in the framework. However, the package does maintain consistent 
conventions across classes to support a range of common usage options. When meaningful, 
each synchronizer supports:

• Nonblocking synchronization attempts (for example, tryLock) as well as blocking versions.
• Optional timeouts, so applications can give up waiting.
• Cancellability via interruption, usually separated into one version of acquire 
that is cancellable, and one that isn't.

Synchronizers may vary according to whether they manage only exclusive states – those 
in which only one thread at a time may continue past a possible blocking point – versus 
possible shared states in which multiple threads can at least sometimes proceed. 
Regular lock classes of course maintain only exclusive state, but counting semaphores, 
for example, may be acquired by as many threads as the count permits. 
To be widely useful, the framework must support both modes of operation.
The java.util.concurrent package also defines interface Condition, supporting monitor-style 
await/signal operations that may be associated with exclusive Lock classes, 
and whose implementations are intrinsically intertwined with their associated Lock classes.
</pre>
</details>

##### 译文
同步器执行两种类型的方法: 至少一个`acquire`来阻塞