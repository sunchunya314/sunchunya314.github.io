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
然而, 这么做必定会带来大量的复杂度, 开销, 以及没有扩展性, 这是一个二流的工程选择.
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
同步器执行两种类型的方法: 至少一个`acquire`来阻塞调用线程直到同步状态允许它继续执行, 至少一个`release`操作来改变同步状态
来允许一个或多个被阻塞线程解锁.
`java.util.concurrent`并发包没有为同步器定义一个统一标准的API. 一些是通过通用的接口(例如`Lock`)来定义, 但是其他的包括专门的版本.
所以, `acquire` 和 `release`操作在不同的类里有不同形式的名字. 例如, 方法`Lock.lock`, `Semaphore.acquire`, 
`CountDownLatch.await`,`FutureTask.get` 都对应着框架的`acquire`操作. 然而, 这个包并没有为不同的类维护一个一致的协议来支持
一定范围的通用使用选项范围. 有意义的是, 每个同步器支持如下三点:
- 非阻塞的同步尝试(例如, `tryLock`)以及阻塞版本.
- 可选的超时, 应用可以放弃等待.
- 可通过中断实现取消, 通常分离成两个版本的`acquire`, 一个可以取消, 一个不可以.

同步器状态管理有互斥和共享两种模式, 互斥模式下一次只能允许一个线程通过阻塞点, 共享模式下一次可以通过多个线程.
通常情况下一个锁类只维护互斥状态, 除了计数信号量可能被很多线程`acquire`作为允许通过的数量.
为了兼容大部分场景, 框架必须同时支持这两种模式. 

`java.util.concurrent`并发包也定义了`Condition`接口, 支持互斥锁的监控器式的`await`/`signal`操作,
这些实现是内聚在它们对应的锁类里.

 
### 2.2 绩效目标

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>2.2 Performance Goals</h3>
Java built-in locks (accessed using synchronized methods and blocks) have long been a performance concern, 
and there is a sizable literature on their construction (e.g., [1], [3]). However, the main focus of such 
work has been on minimizing space overhead (because any Java object can serve as a lock) and on minimizing 
time overhead when used in mostly-single-threaded contexts on uniprocessors. Neither of these are 
especially important concerns for synchronizers: Programmers construct synchronizers only when needed, 
so there is no need to compact space that would otherwise be wasted, and synchronizers are used almost 
exclusively in multithreaded designs (increasingly often on multiprocessors) under which at least 
occasional contention is to be expected. So the usual JVM strategy of optimizing locks primarily 
for the zero-contention case, leaving other cases to less predictable "slow paths" [12] 
is not the right tactic for typical multithreaded server applications that rely heavily on 
java.util.concurrent. 

Instead, the primary performance goal here is scalability: to predictably 
maintain efficiency even, or especially, when synchronizers are contended. Ideally, the overhead 
required to pass a synchronization point should be constant no matter how many threads are trying to 
do so. Among the main goals is to minimize the total amount of time during which some thread is permitted to 
pass a synchronization point but has not done so. However, this must be balanced against resource considerations, 
including total CPU time requirements, memory traffic, and thread scheduling overhead. For example, 
spinlocks usually provide shorter acquisition times than blocking locks, but usually waste cycles and 
generate memory contention, so are not often applicable.

These goals carry across two general styles of use. Most applications should maximize aggregate throughput, 
tolerating, at best, probabilistic guarantees about lack of starvation. However in applications such as 
resource control, it is far more important to maintain fairness of access across threads, tolerating 
poor aggregate throughput. No framework can decide between these conflicting goals on behalf of users; 
instead different fairness policies must be accommodated.

No matter how well-crafted they are internally, synchronizers will create performance bottlenecks in some 
applications. Thus, the framework must make it possible to monitor and inspect basic operations to allow users 
to discover and alleviate bottlenecks. This minimally (and most usefully) entails providing a way to determine 
how many threads are blocked.
</pre>
</details>

##### 译文