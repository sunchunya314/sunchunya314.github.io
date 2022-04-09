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

作者: Doug Lea

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

Java团队根据Java社区流程(JCP)Java特别请求(JSR)166发布J2SE-1.5时引入了 
`java.uti.concurrent`并发集合包, 一个中间件基本的并发支持类. 这些组件是
一个同步器的集合, 抽象数据类型(ADT)类维护一个内部的同步状态(例如, 代表
一把锁是锁住还是未锁住),操作来更新或者监控这个状态, 并且当状态需要的时候至少
提供一个方法用来调用线程阻塞, 当其他线程改变这个这个同步状态时可以唤醒.例子
包括各种形式的互斥锁, 读写锁, 信号量, 屏障, 未来任务, 事件指示, 以及传递队列.

正如已知的几乎所有同步器都可以用来实现其他的同步器. 例如, 可以使用一个可重入锁
可以构建信号量, 反之亦然.
然而, 这么做必定会带来大量的复杂度, 开销, 以及没有扩展性, 这是一个二流的工程选择.
进一步讲, 这么做概念上毫无吸引力. 如果这些结构本质上并没有比其他的更原生, 
那么开发们不应该强行的选择它们中的一个来作为构建其他的基础.
作为代替, JSR166以 `AbstractQueuedSynchronizer` 构建了一个轻量框架, 
为并发包里的同步器提供了通用的机制, 以及一些其他的用户可以自定义的类.

本论文的主旨是讨论这个框架的需求, 设计和实现的主要思想, 简单的用法, 以及对它的
性能指数的一些测量.

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

### 2.1 功能

同步器执行两种类型的方法: 至少一个`acquire`来阻塞调用线程直到同步状态允许它继续执行, 
至少一个`release`操作来改变同步状态来允许一个或多个被阻塞线程解锁.
`java.util.concurrent`并发包没有为同步器定义一个统一标准的API. 一些是通过通用的接口
(例如`Lock`)来定义, 但是其他的包括专门的版本.

所以, `acquire` 和 `release`操作在不同的类里有不同形式的名字. 例如, 方法`Lock.lock`, 
`Semaphore.acquire`, `CountDownLatch.await`,`FutureTask.get` 都对应着框架的
`acquire`操作. 然而, 这个包并没有为不同的类维护一个一致的协议来支持一定范围的通用使用
选项范围. 

有意义的是, 每个同步器支持如下三点:
- 非阻塞的同步尝试(例如, `tryLock`)以及阻塞版本.
- 可选的超时, 应用可以放弃等待.
- 可通过中断实现取消, 通常分离成两个版本的`acquire`, 一个可以取消, 一个不可以.

同步器状态管理有互斥和共享两种模式, 互斥模式下一次只能允许一个线程通过阻塞点, 共享模式
下一次可以通过多个线程.通常情况下一个锁类只维护互斥状态, 除了计数信号量可能被很多线程
`acquire`作为允许通过的数量.为了兼容大部分场景, 框架必须同时支持这两种模式. 

`java.util.concurrent`并发包也定义了`Condition`接口, 支持互斥锁的监控器式的
`await`/`signal`操作, 这些实现是内聚在它们对应的锁类里.

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

### 2.2 性能目标

Java 内置锁(通过`synchronized`关键字锁住方法和代码块)一直有性能问题, 并且它们的
实现结构有很多的问题. 然而, 这个工作的主要焦点是在最小化空间开销(因为任何java对象
都可以作为一个锁对象)和最小化时间开销, 当在单处理器以单线程为主的上下文中使用. 
下面这些问题对同步器都不是主要问题:
程序员只在需要时构建同步器, 所以没有必要压缩可能会浪费的空间, 
一个同步器在多线程设计(多处理器中更容易发生)中主要是互斥模式, 会有竞争出现.
所以JVM对内置锁的优化策略主要在`零-竞争`的场景, 
对于其他的场景无法预测的`慢路径`处理策略主要依赖`java.util.concurrent`并发包.

于是, 这里主要的性能目标是可量测性: 甚至是可预见性的维护效率, 或者特别是, 当同步器被竞争的时候.
理想情况下, 通过同步点的开销应该是一个与线程数无关的常量. 
其中的主要目标是最小化线程从允许有通过一个同步点到通过的持续时间总量.
然而, 这个必须会资源因素平衡, 包括总的CPU时间需求, 内存流量, 以及线程调度开销.
例如, 自旋锁通常比阻塞锁的时间需求更短, 但是会浪费循环和需要更多的内存竞争, 所以不经常使用.

这些目标通过两种使用方式达到. 大部分的应用一个个最大化总吞吐量, 能容忍(最好避免)饥饿. 
然而在资源控制的应用中, 维护访问的线程的公平性更重要, 容忍较低的吞吐量. 
没有框架可以在这些目标冲突中替用户决定选择哪种方案.
框架必须能适应不同的公平策略.

无论内部是多么精心的设计, 在某些应用中同步器都会存在性能瓶颈.
此外, 框架必须允许监控和观察基本操作让用户发现和减轻瓶颈.
这最小限度（也是最有用的）需要提供一种方法来确定有多少线程被阻塞。

 
## 3. 设计和实现

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h2>3. DESIGN AND IMPLEMENTATION</h2>
The basic ideas behind a synchronizer are quite straightforward.

An acquire operation proceeds as:
<code>
    while (synchronization state does not allow acquire) { 
        enqueue current thread if not already queued; 
        possibly block current thread;
    }
    dequeue current thread if it was queued;
</code>

And a release operation is:
<code>    
    update synchronization state;
    if (state may permit a blocked thread to acquire)
        unblock one or more queued threads;
</code>
Support for these operations requires the coordination of three basic components:
• Atomically managing synchronization state
• Blocking and unblocking threads
• Maintaining queues

It might be possible to create a framework that allows each of these 
three pieces to vary independently. 
However, this would neither be very efficient nor usable. For example, 
the information kept in queue nodes must mesh with that needed for 
unblocking, and the signatures of exported methods 
depend on the nature of synchronization state.
The central design decision in the synchronizer framework was to choose 
a concrete implementation of each of these three components, while still 
permitting a wide range of options in how they are used. 
This intentionally limits the range of applicability, but provides 
efficient enough support that there is practically never a reason not 
to use the framework (and instead build synchronizers from scratch) in 
those cases where it does apply.

</pre>
</details>



同步器的背后的基本的理念非常直接. `acquire`操作执行流程如下:

```
    while(没有acquire到同步状态){
        if(当前线程不在队列) 线程加入等待队列;
        阻塞当前线程;
    }
    
    当前线程出队;
```

`release`操作执行流程如下:

```
    更新同步状态;

    if(状态允许一个阻塞线程来acquire){
        唤醒一个或者多个阻塞排队线程;
    }
```

实现以上操作需要3个基本组件合作:

- 原子性的管理同步状态
- 阻塞唤醒线程
- 维护等待队列

获取有可能创建一个允许三个组件单独变化的框架.
然而, 这样做既效率低, 可用性又差.
例如, 保存在队列节点里的信息必须和`唤醒`紧密配合, 并且导出方法的签名基于同步状态的特性.
同步器框架的中心的设计思想是给这三个组件选择一个具体的实现, 同时对于如何使用依然允许一个广泛的选择.
这样做限制了适应的范围, 但是提供了足够高效的支持, 实际生产遇到的场景中没有任何
理由不使用这个框架(并且自己从零构建同步器).

### 3.1 同步状态

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>3.1 Synchronization State</h3>
Class AbstractQueuedSynchronizer maintains synchro- nization state using 
only a single (32bit) int, and exports getState, setState, and compareAndSetState 
operations to access and update this state. These methods in turn rely on 
java.util.concurrent.atomic support providing JSR133 (Java Memory Model) 
compliant volatile semantics on reads and writes, and access to native 
compare-and-swap or load- linked/store-conditional instructions to implement 
compare- AndSetState, that atomically sets state to a given new value 
only if it holds a given expected value.

Restricting synchronization state to a 32bit int was a pragmatic decision. 
While JSR166 also provides atomic operations on 64bit long fields, these 
must still be emulated using internal locks on enough platforms that the 
resulting synchronizers would not perform well. 
In the future, it seems likely that a second base class, specialized for 
use with 64bit state (i.e., with long control arguments), will be added. 
However, there is not now a compelling reason to include it in the package. 
Currently, 32 bits suffice for most applications. Only one java.util.concurrent 
synchronizer class, CyclicBarrier, would require more bits to maintain state, 
so instead uses locks (as do most higher-level utilities in the package).

Concrete classes based on AbstractQueuedSynchronizer must define methods 
tryAcquire and tryRelease in terms of these exported state methods in order to 
control the acquire and release operations. The tryAcquire method must return 
true if synchronization was acquired, and the tryRelease method 
must return true if the new synchronization state may allow future acquires. 
These methods accept a single int argument that can be used to communicate 
desired state; for example in a reentrant lock, to re-establish the recursion 
count when re-acquiring the lock after returning from a condition wait. 
Many synchronizers do not need such an argument, and so just ignore it.
</pre>
</details>


`AbstractQueuedSynchronizer`类同步器只使用一个(32bit)int值维护同步状态, 并且提供`getState`, 
`setState`, 和`compareAndSetState`操作来访问和更新这个状态. 这些方法依靠`java.util.concurrent.atomic`
原子包的支持, 由JSR133(Java内存模型)读写一致性`volatile`机制, 和使用原生方法`compare-and-swap` 和
`load-linked/store-conditional` 指令来实现`compareAndSetState`, 这个方法可以原子性修改`state`的
值只有当它持有一个给予的期望值.

限制同步状态的值为32bit的int类型是根据实际情况决定的.
JSR166对于64bit的long类型自动也提供了原子性的操作, 在很多平台上这个必须使用一个内部锁模仿实现, 使得同步器的
性能不太好.
将来, 会提供第二个基类, 专门为使用64bit的state.
然而, 现在没有一个令人信服的理由把它加入到包里.
目前, 对于大部分应用32bit足够了. `java.util.concurrent`包下只有一个同步器类,`CyclicBarrier`需要更多的bit位
来维护`state`, 所以使用锁代替(在包里完成大部分的上层的使用).

基于`AbstractQueuedSynchronizer`的实现类必须定义方法`tryAcquire`和`tryRelease`, 
根据这些导出的状态方法来控制
`acquire`和`release`操作. 当同步被请求成功时, `tryAcquire`方法必须返回`true`, 如果一个新的同步状态允许
将来的`acquire`那么`tryRelease`必须返回`true`. 这些方法接受一个`int`参数用来交流想要的`state`; 例如,
在可重入锁, 当再次获取锁在从一个`condition wait`返回后统计递归数.
许多同步器不需要这个参数, 只需要忽略它.

### 3.2 阻塞
<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>3.2 Blocking</h3>
Until JSR166, there was no Java API available to block and unblock threads 
for purposes of creating synchronizers that are not based on built-in monitors. 
The only candidates were Thread.suspend and Thread.resume, which are unusable 
because they encounter an unsolvable race problem: If an unblocking thread 
invokes resume before the blocking thread has executed suspend, the resume 
operation will have no effect.

The java.util.concurrent.locks package includes a LockSup- port class with 
methods that address this problem. Method LockSupport.park blocks the current 
thread unless or until a LockSupport.unpark has been issued. (Spurious wakeups 
are also permitted.) Calls to unpark are not "counted", so multiple unparks 
before a park only unblock a single park. Additionally, this applies per-thread, 
not per-synchronizer. A thread invoking park on a new synchronizer might return 
immediately because of a "leftover" unpark from a previous usage. However, in 
the absence of an unpark, its next invocation will block. While it would be 
possible to explicitly clear this state, it is not worth doing so. It is more 
efficient to invoke park multiple times when it happens to be necessary.

This simple mechanism is similar to those used, at some level, in the Solaris-9 
thread library [11], in WIN32 "consumable events", and in the Linux NPTL thread 
library, and so maps efficiently to each of these on the most common platforms 
Java runs on. (However, the current Sun Hotspot JVM reference implementation on 
Solaris and Linux actually uses a pthread condvar in order to fit into the 
existing runtime design.) The park method also supports optional relative and 
absolute timeouts, and is integrated with JVM Thread.interrupt 
support — interrupting a thread unparks it.

</pre>
</details>

JSR166之前, 没有Java API可以阻塞和唤醒线程用于创建非基于内嵌锁的同步器.
唯一的候选人是`Thread.suspend`和`Thread.resume`, 但是根本没用, 因为
它们遇到了一个无法解决的竞太问题: 如果一个没有阻塞线程在阻塞线程被挂起之前
调用唤醒, 这个\
唤醒操作<font color='red'>无效</font>.

`java.util.concurrent.locks`包里有一个`LockSupport`类提供了方法解决这个问题.
方法 `LockSupport.park` 阻塞当前线程直到调用`LockSupport.unpark`. 
(伪造的唤醒也可以.)调用`unpark`不会被"计数", 所以在`park`之前多次`unpark`,只会
`unlock`一个`park`.此外, 这个作用于线程而不是同步器. 一个线程在一个新的同步器里
调用`park`可能会立刻返回由于先前"残留"的`unpark`. 然而, 没有`unpark`, 下次调用
将会阻塞. 虽然它可以显式的清空这个状态, 但不值得这么做. 当它恰巧需要的时候, 多次
调用`park`更高效.

在某种程度上, 这个简单的机制和在`Solaris 9`线程库, `WIN32` 可消费事件, Linux NPTL 线程库
使用相似, 在大部分Java运行的平台上可以高效的映射. (然而, 当前Sun Hotspot JVM 在Solaris 
和 Linux引用实现是使用一个pthread condvar 为了适应已有的运行设计.) `park`方法通用支持选择
相对时间和绝对时间超时, 并且完美的支持JVM `Thread.interrupt`方法 - 中断一个线程唤醒它.

### 3.3 队列

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>3.3 Queues</h3>

The heart of the framework is maintenance of queues of blocked threads, which are 
restricted here to FIFO queues. Thus, the framework does not support priority-based 
synchronization.

These days, there is little controversy that the most appropriate choices for 
synchronization queues are non-blocking data structures that do not themselves 
need to be constructed using lower-level locks. And of these, there are two main 
candidates: variants of Mellor-Crummey and Scott (MCS) locks [9], and variants of 
Craig, Landin, and Hagersten (CLH) locks [5][8][10]. Historically, CLH locks have 
been used only in spinlocks. However, they appeared more amenable than MCS for use 
in the synchronizer framework because they are more easily adapted to handle 
cancellation and timeouts, so were chosen as a basis. The resulting design is far 
enough removed from the original CLH structure to require explanation.

A CLH queue is not very queue-like, because its enqueuing and dequeuing 
operations are intimately tied to its uses as a lock. It is a linked queue 
accessed via two atomically updatable fields, head and tail, both initially 
pointing to a dummy node.

A new node, node, is enqueued using an atomic operation: 
<code>
    do { 
        pred = tail;
    } while(!tail.compareAndSet(pred, node));
</code>
The release status for each node is kept in its predecessor node. 
So, the "spin" of a spinlock looks like:
<code>
    while (pred.status != RELEASED) ; // spin
    A dequeue operation after this spin simply entails setting the head 
</code>    
field to the node that just got the lock:
<code>    
    head = node;
</code>
Among the advantages of CLH locks are that enqueuing and dequeuing are fast, 
lock-free, and obstruction free (even under contention, one thread will always 
win an insertion race so will make progress); that detecting whether any threads 
are waiting is also fast (just check if head is the same as tail); and that release 
status is decentralized, avoiding some memory contention.

In the original versions of CLH locks, there were not even links connecting nodes. 
In a spinlock, the pred variable can be held as a local. However, Scott and Scherer[10] 
showed that by explicitly maintaining predecessor fields within nodes, CLH locks can 
deal with timeouts and other forms of cancellation: If a node's predecessor cancels, 
the node can slide up to use the previous node's status field.

The main additional modification needed to use CLH queues for blocking synchronizers 
is to provide an efficient way for one node to locate its successor. In spinlocks, 
a node need only change its status, which will be noticed on next spin by its successor, 
so links are unnecessary. But in a blocking synchronizer, a node needs to explicitly 
wake up (unpark) its successor.

An AbstractQueuedSynchronizer queue node contains a next link to its successor. 
But because there are no applicable techniques for lock-free atomic insertion of 
double-linked list nodes using compareAndSet, this link is not atomically set as 
part of insertion; it is simply assigned:
pred.next = node;
after the insertion. This is reflected in all usages. The next link is treated only 
as an optimized path. If a node's successor does not appear to exist (or appears to 
be cancelled) via its next field, it is always possible to start at the tail of the 
list and traverse backwards using the pred field to accurately check if there really 
is one.

A second set of modifications is to use the status field kept in each node for purposes 
of controlling blocking, not spinning. In the synchronizer framework, a queued thread 
can only return from an acquire operation if it passes the tryAcquire method defined 
in a concrete subclass; a single "released" bit does not suffice. But control is still 
needed to ensure that an active thread is only allowed to invoke tryAcquire when it 
is at the head of the queue; in which case it may fail to acquire, and (re)block. 
This does not require a per-node status flag because permission can be determined by 
checking that the current node's predecessor is the head. And unlike the case of 
spinlocks, there is not enough memory contention reading head to warrant replication. 
However, cancellation status must still be present in the status field.

The queue node status field is also used to avoid needless calls to park and unpark. 
While these methods are relatively fast as blocking primitives go, they encounter 
avoidable overhead in the boundary crossing between Java and the JVM runtime and/or OS. 
Before invoking park, a thread sets a "signal me" bit, and then rechecks synchronization 
and node status once more before invoking park. A releasing thread clears status. 
This saves threads from needlessly attempting to block often enough to be worthwhile, 
especially for lock classes in which lost time waiting for the next eligible thread 
to acquire a lock accentuates other contention effects. This also avoids requiring 
a releasing thread to determine its successor unless the successor has set the signal 
bit, which in turn eliminates those cases where it must traverse multiple nodes to 
cope with an apparently null next field unless signalling occurs in conjunction 
with cancellation.

Perhaps the main difference between the variant of CLH locks used in the synchronizer 
framework and those employed in other languages is that garbage collection is relied 
on for managing storage reclamation of nodes, which avoids complexity and overhead. 
However, reliance on GC does still entail nulling of link fields when they are sure 
to never to be needed. This can normally be done when dequeuing. Otherwise, unused 
nodes would still be reachable, causing them to be uncollectable.

Some further minor tunings, including lazy initialization of the initial dummy node 
required by CLH queues upon first contention, are described in the source code 
documentation in the J2SE1.5 release.
    
Omitting such details, the general form of the resulting implementation of the 
basic acquire operation (exclusive, noninterruptible, untimed case only) is:
<code>
    if (!tryAcquire(arg)) {
        node = create and enqueue new node;
        pred = node's effective predecessor;
        while (pred is not head node || !tryAcquire(arg)) {
            if (pred's signal bit is set)
                park();
            else
                compareAndSet pred's signal bit to true;
            pred = node's effective predecessor; 
        }
        head = node;
    }    
</code>
And the release operation is:
<code>
    if (tryRelease(arg) && head node's signal bit is set) { 
        compareAndSet head's signal bit to false;
        unpark head's successor, if one exists;
    }
</code>    
The number of iterations of the main acquire loop depends, of course, on the 
nature of tryAcquire. Otherwise, in the absence of cancellation, each component 
of acquire and release is a constant-time O(1) operation, amortized across 
threads, disregarding any OS thread scheduling occuring within park.
Cancellation support mainly entails checking for interrupt or timeout upon 
each return from park inside the acquire loop. A cancelled thread due to 
timeout or interrupt sets its node status and unparks its successor so it 
may reset links. With cancellation, determining predecessors and successors 
and resetting status may include O(n) traversals (where n is the length of 
the queue). Because a thread never again blocks for a cancelled operation, 
links and status fields tend to restabilize quickly.

</pre>
</details>

框架的心脏是阻塞线程队列的维护, 这里限定为`FIFO`队列. 此外, 框架不支持基于优先级的同步.

这些天, 关于同步队列最合适的选择有一些争论, 同步队列是不需要使用低级别锁的非阻塞数据机构.
基于此, 有两个主要的候选人: MCS锁变体, CLH锁变体. 历史上, CLH锁只在自旋锁中使用. 然而.
在同步器框架中它们比MCS更容易控制, 因为它们更容易适应处理取消和超时, 所以被选做基础. 
所产生的设计与原始的CLH结构非常遥远，需要进行解释。

一个CLH队列不是很像队列, 因为他的入队和出队操作与它作为锁的用途紧密相连. 它是一个链表队列,
通过两个原子更新的字段访问, 头和尾, 初始化时都是指向一个虚拟节点.

![图片](https://scyblog.oss-cn-shanghai.aliyuncs.com/2022/04/04/node_status.jpg)

一个新的节点, 入队采用原子操作:
```
    do{
        pred = tail;
    } while(!tail.comapreAndSet(pred,node));
```

每个节点的释放状态都保存在它前一个节点里.
所以, 自旋锁的`自旋`看起来像:
```
    while(pred.status != RELEASED); //spin
```
自旋之后的出队操作只需要设置头字段为刚持有这把锁的节点:
```groovy
    head = code;
```

CLH锁的优势是入队和出队操作快速, 无锁, 无阻塞(即使有竞争, 总有一个赢得插入竞赛的线程执行);
监测是否存在等待线程快速(只需要检查头是否等于尾);并且是否状态是去中心化的, 避免内存竞争.

CLH锁的原始版本里, 节点之间没有链接. 在自旋锁, 前节点变化可以当做本地变量.
然而, Scott 和 Scherer 展示通过显式维护前节点的字段, CLH锁可以处理超时和其他形式的取消:
如果一个节点的前节点取消了, 那么节点可以向前滑动以使用上一个节点的状态字段.

用CLH队列作为阻塞同步器的主要的改动是提供一个高效的方式来定位一个节点的后继节点.
在自旋锁里, 一个节点只需要改变它的状态, 就会被它的后继节点在下一次自旋里发现.
所以连接是没有必要的. 但是在阻塞队列同步器里, 一个节点需要显式的唤醒它的后继节点.

#### 感悟
*(这里终于明白了AQS对CLH做了什么改动: 原来的CLH队列, 用的是自旋锁,
<font bold='true' color='blue'> 节点之间没有`next`连接</font>, 只有`pred`链接, 一个节点释放锁, 
不需要显式通知它的下一个节点, 只需要改变自己的状态, 那么下一个节点会自旋中发现前节点的状态改变,
从而可以获取锁. 但是AQS里用的的是阻塞线程, 一个节点释放, 后继节点处于阻塞中无法主动检测到前节点的状态改变,
所以需要`LockSupport.unpark(node.next.thread)`来显式的唤醒后继节点.)*


一个AbstractQueuedSynchronizer队列节点包含一个指向后继节点的`next`链接.
但是因为那里没有合适的技术通过`compareAndSet`来实现双向链表的无锁原子插入, 
所以这个链接在插入操作时并不是原子性的设置; 它只是简单赋值:
```
    pred.next=node;
```
插入之后. 它反映到所有的使用中. `next`链接只是作为一个优化路径. 如果一个节点的
后继节点通过它的`next`字段来访问时不存在(或者被取消), 它将会从尾节点开始反向遍历, 
使用`pred`字段来精确的检查是否存在后继节点.

#### 源码解读: 插入节点和寻找后继节点
1. 插入节点部分代码:
```groovy
    node.prev = pred; // 1 设置节点pred链接
    if (compareAndSetTail(pred, node)) { // 2 设置tail节点
        pred.next = node; // 3 设置pred节点的next链接
    }
```
插入操作一共三行代码, 当第2行执行成功时, 这个节点已经实质上插入到队列里, 但是
当第三行还没有执行的时候, 其他线程使用`pred.next`访问`pred`节点的后继节点时就会为`null`,
所以插入操作中`next`链接的设置并不是原子操作. 但是`pred`和`tail`的设置是原子操作,
所以更精确的查找后继节点的方法是, 从尾节点开始反向遍历. 源码也是这么实现的:
2. 寻找后继节点代码:
```groovy
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 反向遍历到当前节点为止(不包括当前)
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0) // 寻找没有取消的后继节点
                    s = t;
        }
```

第二个改动是每个节点保存一个状态字段用来控制阻塞, 而不是自旋. 在同步器框架, 一个入队的线程
只能从一个`acquire`操作返回, 如果它通过了子类定义的`tryAcquire`方法; 单一个`release`还不够.
但是控制依然需要, 以确保一个活动的线程只有它处于队列的最前面时调用`tryAcquire`;
如果`acquire`失败, 会再次阻塞.这并不需要每个节点的状态标记, 因为可以通过检查当前节点
的前节点是否是头来确定权限. 不同于自旋锁场景, 这里没有足够的内存争用读取头来保证复制.??
然而, 取消状态依然在状态字段中.

队列节点状态字段也用来避免不必要的`park`和`unpark`调用.
这些方法相对于原生的阻塞要更快, 它们避免了在Java和JVM运行时/或操作系统的边界切换开销.
在调用`park`之前, 一个线程设置一个`唤醒我`bit, 然后再次检查同步和节点状态,再调用
`park`之前再来一次. 
一个释放线程清空状态. 这使得线程不必努力阻塞得足够有价值, 特别是对于锁类, 等待下一个合格
线程来获取锁的时间损失严重受到竞争影响. 这个也避免了需要一个释放线程来确定它的后继者, 除非
后继者已经设置了信号位, 反过来这也消除了那些它必须遍历多个节点来处理`next`字段为空的情况, 
除非信号和取消同时产生.

也许同步器框架中使用的CLH锁的变体和其他语言中使用的主要区别在于依赖垃圾收集来管理节点
的内存回收，这避免了复杂性和开销.
当然当确定链接字段不再需要时, 依赖垃圾回收依然需要确保它们被设为null.
这通常可以在出队时完成. 否则, 无用的节点依然可以到达, 使它们无法回收.

一些进一步的优化, 包括CLH队列第一竞争时需要的虚拟节点的懒初始化, 在J2SE1.5发布的源码文档
作了描述.

忽略这些细节, 基础的`acquire`操作最终实现的一般形式(仅限互斥, 不中断, 无超时情况)是:
```groovy
   if(tryAcquire(arg)){
        node = new Node();
        enqueue(node);
        pred = node.predecessor();
        while(pred != head || !tryAcquire(arg)){
            if(pred.ws==SIGNAL){// 前节点的ws(waitStatus)值是否SIGNAL 
                park();
            }else{
                // 这里先不park, 再跑一次循环, 尝试acquire, 如果失败就会park   
                // 看源码时, 我们会好奇SIGNAL这个值是什么时候第一次设置的, 答案就是在这里, 目的是下一次循环park
                // 问题: 为什么这里设置完 SIGNAL不直接park? 
                // 答案: 因为release操作里, 只有头结点ws值是SIGNAL才会唤醒后继节点, 那么如果这行代码还未执行, 
                // release将会错过唤醒后继节点的机会. 如果这个节点在这里park, 那么将会永远沉睡, 没有人来唤醒.                        
                compareAndSet(pred.ws, SIGNAL); 
            }
            pred = node.predecessor();
        }                                                                                                                                                                                                         
   }

```
 `release` 操作是:
```groovy
    if (tryRelease(arg) && head.ws == SIGNAL) { // 释放成功后, 必须头结点的ws值为SIGNAL才会唤醒后继节点
        compareAndSet(head.ws,0); // 清空头结点的等待状态ws
        unpark(head.successor()) // 唤醒后继节点
    }
```

当然, 主`acquire`循环迭代的次数取决于`tryAcquire`的性质. 否则, 在没有取消
的情况下, 不考虑在`park`时发生的任何系统调用发生的消耗, `acquire`和`release`
的每个组件都是常量时间O(1)操作. 
支持取消主要是确保在`acquire`循环内部每次从`park`返回检查中断和超时.
一个由于超时或者中断取消的线程会修改它的节点状态并且唤醒它的后继节点, 
所以它可能会重置链接. 当取消时, 确定前节点和后继节点并重置状态可能包括O(n)遍历
(n是队列的长度).
因为线程不会再阻塞已取消的操作, 所以链接和状态字段往往会迅速重新稳定.

### 3.4 条件队列

<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>3.4 Condition Queues</h3>

The synchronizer framework provides a ConditionObject class for use 
by synchronizers that maintain exclusive synchronization and conform 
to the Lock interface. Any number of condition objects may be 
attached to a lock object, providing classic monitor-style await, 
signal, and signalAll operations, including those with timeouts, 
along with some inspection and monitoring methods.

The ConditionObject class enables conditions to be efficiently 
integrated with other synchronization operations, again by fixing 
some design decisions. This class supports only Java-style monitor 
access rules in which condition operations are legal only when the 
lock owning the condition is held by the current thread (See [4] 
for discussion of alternatives). Thus, a ConditionObject attached 
to a ReentrantLock acts in the same way as do built-in monitors 
(via Object.wait etc), differing only in method names, 
extra functionality, and the fact that users can declare multiple 
conditions per lock.

A ConditionObject uses the same internal queue nodes as 
synchronizers, but maintains them on a separate condition queue. 
The signal operation is implemented as a queue transfer from 
the condition queue to the lock queue, without necessarily 
waking up the signalled thread before it has re-acquired its lock.

The basic await operation is:
<code>
create and add new node to condition queue; 
release lock;
block until node is on lock queue;
re-acquire lock;
</code>
And the signal operation is:
<code>
transfer the first node from condition queue to lock queue;
</code>
Because these operations are performed only when the lock is held, 
they can use sequential linked queue operations (using a 
nextWaiter field in nodes) to maintain the condition queue. 
The transfer operation simply unlinks the first node from the 
condition queue, and then uses CLH insertion to attach it to 
the lock queue.

The main complication in implementing these operations is dealing
with cancellation of condition waits due to timeouts or 
Thread.interrupt. A cancellation and signal occuring at 
approximately the same time encounter a race whose outcome conforms 
to the specifications for built-in monitors. As revised in JSR133, 
these require that if an interrupt occurs before a signal, then the 
await method must, after re-acquiring the lock, throw 
InterruptedException. But if it is interrupted after a signal, 
then the method must return without throwing an exception, 
but with its thread interrupt status set.

To maintain proper ordering, a bit in the queue node status 
records whether the node has been (or is in the process of being) 
transferred. Both the signalling code and the cancelling code try 
to compareAndSet this status. If a signal operation loses this race, 
it instead transfers the next node on the queue, if one exists. 
If a cancellation loses, it must abort the transfer, and then await 
lock re-acquisition. This latter case introduces a potentially 
unbounded spin. A cancelled wait cannot commence lock re- acquisition 
until the node has been successfully inserted on the lock queue, 
so must spin waiting for the CLH queue insertion compareAndSet being 
performed by the signalling thread to succeed. The need to spin 
here is rare, and employs a Thread.yield to provide a scheduling 
hint that some other thread, ideally the one doing the signal, 
should instead run. While it would be possible to implement here 
a helping strategy for the cancellation to insert the node, 
the case is much too rare to justify the added overhead that this 
would entail. In all other cases, the basic mechanics here and 
elsewhere use no spins or yields, which maintains reasonable 
performance on uniprocessors.
</pre>
</details>

同步器框架提供了一个`ConditionObject`类, 供维护互斥同步的同步器使用,
并且遵循`Lock`接口. 任意数量的条件对象可以被附加到锁对象上, 提供经典的
监控器式的等待, 唤醒, 和唤醒全部操作, 包括超时, 以及一些监控和检测的方法.

`ConditionObject`类再次通过修复一些设计决策, 使条件与其他同步操作有效的
集成. 此类只支持java类型的监控器访问规则, 其中只有当拥有该条件的锁被
当前线程持有时, 条件操作才是合法的. 因此, 附加到`ReentrantLock`重入锁的
`ConditionObject`条件对象和内置监控器的操作相同, 区别只有方法名,
额外的功能, 事实是每把锁用户可以声明多个条件.

`ConditionObject`和同步器使用同样的内部队列节点, 但是使用一个不同的
条件队列维护它们. 信号操作被实现为从条件队列到锁队列的队列传输, 在被从条件
队列移出的线程在它再次获取到锁之前不需要被解除阻塞.(疑问: 不解除阻塞怎么抢锁?)
基本等待操作是:
```
创建一个新节点并添加到条件队列;
释放锁;
阻塞该节点直到回到锁队列;
再次获取锁.
``` 
  
唤醒操作是:
```
将条件队列的第一个节点转移到锁队列;
```

只有当前线程持有锁的时候才能执行这些操作, 它们可以使用按顺序的链表队列操作
(节点里使用一个`nextWaiter`字段)来维护条件队列.
转移操作只需要简单地修改等待队列第一个节点的链接, 然后使用CLH的插入将它加入
到锁队列.

实现这些操作的主要复杂的地方在于处理由于超时或线程中断而引起的条件等待取消.
取消和信号同时发生, 那么会遇到一个符合内置监视器规格的竞塞.
根据JSR133的修订, 要求信号之前发生中断, 那么`await`方法, 再重新获取锁之后,
必须抛出`InterruptedException`异常. 但是如果它是在信号之后中断, 那么该
方法必须返回, 不需要抛出异常, 当是它所在的线程需要设置中断状态.

为了保持正确的顺序, 队列节点状态的一个`bit`记录节点是否(或则已经)转移. 
唤醒代码和取消代码两者都会努力`compareAndSet`这个状态. 如果一个唤醒
操作在竞赛中输了, 它会转移队列里下一个节点作为代替.
如果一个取消操作输了, 它必须禁止此次转移, 然后等待再次获取锁. 后者的场景
会引入一个潜在的无边界自旋. 在节点成功插入到锁队列之前, 一个取消的等待不能
开始获取锁. 所以必须自旋等待唤醒的线程使用`compareAndSet`插入CLH队列成功.
这里很少需要自旋, 并且采用`Thread.yield`来提供一个调度提示其他线程, 理想
的是发信号的线程应该运行. (解读:这里意思是发信号的线程`Thread.yield`并不理想,
而是应该保持运行). 虽然这里可以实现取消插入节点的帮助策略, 但这个场景非常罕见, 无法
证明这将带来的增加开销. 在所有其他的场景下, 不使用自旋或`yields`的基本机制
可以在单处理器上保持合理的性能.                   

## 4. 用法
<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h2>4. USAGE</h2>
Class AbstractQueuedSynchronizer ties together the above functionality 
and serves as a "template method pattern" [6] base class for synchronizers. 
Subclasses define only the methods that implement the state inspections 
and updates that control acquire and release. However, subclasses of 
AbstractQueuedSynchronizer are not themselves usable as synchronizer ADTs, 
because the class necessarily exports the methods needed to internally 
control acquire and release policies, which should not be made visible 
to users of these classes. All java.util.concurrent synchronizer 
classes declare a private inner AbstractQueuedSynchronizer subclass 
and delegate all synchronization methods to it. This also allows public 
methods to be given names appropriate to the synchronizer.

For example, here is a minimal Mutex class, that uses synchronization 
state zero to mean unlocked, and one to mean locked. This class does 
not need the value arguments supported for synchronization methods, 
so uses zero, and otherwise ignores them.

<code>
class Mutex {
    class Sync extends AbstractQueuedSynchronizer {
        public boolean tryAcquire(int ignore) {
            return compareAndSetState(0, 1); 
        }
        public boolean tryRelease(int ignore) {
            setState(0); return true; 
        }
     }
     private final Sync sync = new Sync(); 
        public void lock() { 
     sync.acquire(0); 
     } 
     public void unlock() { 
        sync.release(0); 
     }
}        
</code>

A fuller version of this example, along with other usage guidance 
may be found in the J2SE documentation. Many variants are of course 
possible. For example, tryAcquire could employ "test- and-test-and-set" 
by checking the state value before trying to change it.

It may be surprising that a construct as performance-sensitive 
as a mutual exclusion lock is intended to be defined using a 
combination of delegation and virtual methods. However, these 
are the sorts of OO design constructions that modern dynamic compilers 
have long focussed on. They tend to be good at optimizing away this 
overhead, at least in code in which synchronizers are invoked frequently.

Class AbstractQueuedSynchronizer also supplies a number of 
methods that assist synchronizer classes in policy control. 
For example, it includes timeout and interruptible versions of 
the basic acquire method. And while discussion so far has focussed 
on exclusive-mode synchronizers such as locks, the AbstractQueuedSynchronizer 
class also contains a parallel set of methods (such as acquireShared) 
that differ in that the tryAcquireShared and tryReleaseShared methods 
can inform the framework (via their return values) that further acquires 
may be possible, ultimately causing it to wake up multiple threads 
by cascading signals.

Although it is not usually sensible to serialize (persistently store 
or transmit) a synchronizer, these classes are often used in turn to 
construct other classes, such as thread-safe collections, that are 
commonly serialized. The AbstractQueuedSynchronizer and ConditionObject 
classes provide methods to serialize synchronization state, but not 
the underlying blocked threads or other intrinsically transient bookkeeping. 
Even so, most synchronizer classes merely reset synchronization state 
to initial values on deserialization, in keeping with the implicit 
policy of built-in locks of always deserializing to an unlocked state. 
This amounts to a no-op, but must still be explicitly supported to enable 
deserialization of final fields.
</pre>
</details>

`AbstractQueuedSynchronizer`类将上述功能连接在一起, 并作为同步器的模板方法模式的
基类. 子类只需要定义实现控制获取和释放状态的检查和更新的方法. 然而, `AbstractQueuedSynchronizer`
的子类自身并不用作同步器的抽象数据类型, 因为类必须导出内部控制获取和释放策略的方法, 
不应使其被这些类的使用者可见. `java.util.concurrent`包里所有的同步器类声明了一个
私有的内部`AbstractQueuedSynchronizer`子类并且把所有同步器的方法都委托给它.
这还允许公共方法指定适合同步器的名称.

例如, 这里有一个最简的互斥类, 使用同步状态0代表未锁主, 1代表被锁住. 这个类不需要
给同步方法传参数, 所以使用0, 或者忽略它们.

```java
class Mutex {
    class Sync extends AbstractQueuedSynchronizer {
        public boolean tryAcquire(int ignore) {
            return compareAndSetState(0, 1); 
        }
        public boolean tryRelease(int ignore) {
            setState(0); return true; 
        }
     }
     private final Sync sync = new Sync(); 
        public void lock() { 
     sync.acquire(0); 
     } 
     public void unlock() { 
        sync.release(0); 
     }
}
```

这个案例的完整版, 以及其他的使用指南, 可以J2SE的文档找到. 当然可以有许多的变化.
例如, `tryAcquire`通过在改变`state`值之前检查`state`值实现`test- and-test-and-set`.
令人惊讶的是, 向互斥锁这样对性能敏感的结构会使用委托组合和虚拟方法来定义.
然而, 这些都是现代冬天编译器长期以来一直关注的面向对象动态设计结构. 它们往往擅长优化以消除
这种开销, 至少在经常调用同步器的代码中是这样.

`AbstractQueuedSynchronizer`类在策略控制也提供了很多方法帮助同步器类. 例如, 它包含了
基本`acquire`方法的超时和中断版本. 虽然到目前为止讨论的焦点一直集中在互斥模式的同步器
, 例如锁, `AbstractQueuedSynchronizer`类也包含一组并行的方法(比如`acquireShared`),
不同的`tryAcquireShared`和`tryReleaseShared`方法可以通知框架(通过它们的返回值)来
进一步获取也是可能的, 最终使得它唤醒多个线程通过信号级联.

尽管通常不宜序列化同步器(持续存储和传输), 这些类通过用来构建其他类, 比如线程安全的集合,
经常被序列化. `AbstractQueuedSynchronizer`和`ConditionObject`类提供方法来序列化
同步状态, 但不是潜在的阻塞线程或者瞬态薄记.  即便如此, 大部分的同步器类也只是在反序列化
的时候将同步状态重设为初始值, 为了符合内置锁的隐式策略, 总是反序列化到一个未锁状态.
这相当于没有操作, 但是必须显式支持使得可以反序列化`final`字段.

### 4.1 公平控制
<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>4.1 Controlling Fairness</h3> 
Even though they are based on FIFO queues, synchronizers are not necessarily 
fair. Notice that in the basic acquire algorithm (Section 3.3), tryAcquire 
checks are performed before queuing. Thus a newly acquiring thread can “steal” 
access that is "intended" for the first thread at the head of the queue.

This barging FIFO strategy generally provides higher aggregate throughput than 
other techniques. It reduces the time during which a contended lock is available 
but no thread has it because the intended next thread is in the process of 
unblocking. At the same time, it avoids excessive, unproductive contention by 
only allowing one (the first) queued thread to wake up and try to acquire upon 
any release. Developers creating synchronizers may further accentuate barging 
effects in cases where synchronizers are expected to be held only briefly by 
defining tryAcquire to itself retry a few times before passing back control.

Barging FIFO synchronizers have only probablistic fairness properties. 
An unparked thread at the head of the lock queue has
an unbiased chance of winning a race with any incoming barging thread, 
reblocking and retrying if it loses. However, if incoming threads arrive f
aster than it takes an unparked thread to unblock, the first thread in the 
queue will only rarely win the race, so will almost always reblock, and its 
successors will remain blocked. With briefly-held synchronizers, it is common 
for multiple bargings and releases to occur on multiprocessors during the time 
the first thread takes to unblock. As seen below, the net effect is to maintain 
high rates of progress of one or more threads while still at least probabilistically 
avoiding starvation.

When greater fairness is required, it is a relatively simple matter to arrange it. 
Programmers requiring strict fairness can define tryAcquire to fail (return false) 
if the current thread is not at the head of the queue, checking for this using 
method getFirstQueuedThread, one of a handful of supplied inspection methods.

A faster, less strict variant is to also allow tryAcquire to succeed if the the queue 
is (momentarily) empty. In this case, multiple threads encountering an empty queue may 
race to be the first to acquire, normally without enqueuing at least one of them. 
This strategy is adopted in all java.util.concurrent synchronizers supporting a "fair" 
mode.

While they tend to be useful in practice, fairness settings have no guarantees, 
because the Java Language Specification does not provide scheduling guarantees. 
For example, even with a strictly fair synchronizer, a JVM could decide to run a 
set of threads purely sequentially if they never otherwise need to block waiting for 
each other. In practice, on a uniprocessor, such threads are
likely to each run for a time quantum before being pre-emptively context-switched. 
If such a thread is holding an exclusive lock, it will soon be momentarily switched back, 
only to release the lock and block now that it is known that another thread needs the 
lock, thus further increasing the periods during which a synchronizer is available but 
not acquired. Synchronizer fairness settings tend to have even greater impact on 
multiprocessors, which generate more interleavings, and hence more opportunities 
for one thread to discover that a lock is needed by another thread.

Even though they may perform poorly under high contention when protecting briefly-held 
code bodies, fair locks work well, for example, when they protect relatively long code 
bodies and/or with relatively long inter-lock intervals, in which case barging provides 
little performance advantage and but greater risk of indefinite postponement. 
The synchronizer framework leaves such engineering decisions to its users.

</pre>
</details>

即使它们是基于`FIFO`队列, 同步器不是必须公平. 注意一下基础的`acquire`算法, 在入队之前执
行了`tryAcquire`. 那么一个新来的想要获取的线程可以从队列头部第一个准备获取的线程手里`偷到`
访问权.

这个允许插队的`FIFO`策略主要比其他技术提供更高的聚合吞吐量. 它减少了锁的闲置时间, 即一把
被竞争的锁虽然可以获取但由于下一个线程还在唤醒的过程中导致没有线程持有它. 与此同时, 它只
允许一个(第一个)队列的线程在任何释放的时候醒来并尝试获取, 从而避免了过度的无效的争用.
开发人员可能在创建同步器的时候进一步的增强闯入效果, 只需要在定义`tryAcquire`的时候自身
多试几次, 在返回控制之前.

![](https://scyblog.oss-cn-shanghai.aliyuncs.com/code/barging_thread.png)

插队式先进先出同步器只有一个只有概率性的公平性. 锁队列头部的唤醒线程面对突然插队的线程
有一个无偏差的机会赢得比赛, 输了的话会再次阻塞并重新获取. 然而, 如果闯入的线程到达的速率
比唤醒线程的速度要快, 那么队列里第一个线程很难赢得竞赛, 所以将会一直再次阻塞, 并且它的
后继节点也会一直保持阻塞. 对于简单持有的同步器, 第一个线程解除阻塞的时间内, 在多处理器上
发生插队和释放是很常见的. 如下图所示, 净效应是保持一个或多个线程高执行率, 并尽可能
避免出现饥饿.
 
当需要更高的公平性时, 安排它是一件相对简单的事情. 程序员需要严格的公平性,
使用方法`getFirstQueuedThread`检查, 如果当前线程不在队列的头部
可以定义`tryAcquire`失败(返回`false`).

一个更快, 但不严格的变体时, 如果队列(暂时)为空, 也允许`tryAcquire`成功.
在这个场景下, 多个线程遇到一个空队列可能会竞赛来第一个获取, 通常至少没有一个
人在排队. `java.util.concurrent`并发包里所有的同步器都采用这个策略来
支持`公平性`模式.

虽然它们在实践中很有用, 但公平性设置没有保证, 因为Java语言说明没有提供调度保证.
例如, 即使一个严格公平的同步器, 一个JVM可以纯粹按顺序决定运行一组线程, 如果
它们从来不需要阻塞彼此等待。 在实践中, 但单处理器里, 这样的线程很可能在先发制人的
上下文切换之前, 先运行一个时间片. 如果这样一个线程持有一个互斥锁, 它将很快就会
被切换会, 只有释放这个锁并且阻塞才会直到另一个线程需要那把锁, 从而进一步增加了
同步器可用但没有获取的持续时间. 同步器公平性的设置在产生更多交互的多处理器有更
大的影响, 因此一个线程有更多的机会发现另一个线程需要锁.

即使在保护短暂持有的代码体时，它们可能在高争用下表现不佳，但公平锁工作得很好，
例如，当它们保护相对较长的代码体和/或相对较长的互锁间隔时，在这种情况下，
插队几乎不能提供什么性能优势，但无限期延迟的风险更大. 同步器框架把这个工程
决定留给用户来做.

### 4.2 同步器实现类
<details>
<summary><font size="2" color="grey"><i>原文</i></font></summary>
<pre>
<h3>4.2 Synchronizers</h3> 
Here are sketches of how java.util.concurrent synchronizer classes 
are defined using this framework:
The ReentrantLock class uses synchronization state to hold the 
(recursive) lock count. When a lock is acquired, it also records 
the identity of the current thread to check recursions and detect 
illegal state exceptions when the wrong thread tries to unlock. 
The class also uses the provided ConditionObject, and exports other 
monitoring and inspection methods. The class supports an optional 
"fair" mode by internally declaring two different 
AbstractQueuedSynchronizer subclasses (the fair one disabling barging) 
and setting each ReentrantLock instance to use the appropriate one 
upon construction.

The ReentrantReadWriteLock class uses 16 bits of the synchronization 
state to hold the write lock count, and the remaining 16 bits to hold 
the read lock count. The WriteLock is otherwise structured in the 
same way as ReentrantLock. The ReadLock uses the acquireShared methods 
to enable multiple readers.

The Semaphore class (a counting semaphore) uses the synchronization 
state to hold the current count. It defines acquireShared to decrement 
the count or block if nonpositive, and tryRelease to increment the count, 
possibly unblocking threads if it is now positive.

The CountDownLatch class uses the synchronization state to represent 
the count. All acquires pass when it reaches zero.

The FutureTask class uses the synchronization state to represent the 
run-state of a future (initial, running, cancelled, done). Setting or 
cancelling a future invokes release, unblocking threads waiting for 
its computed value via acquire.

The SynchronousQueue class (a CSP-style handoff) uses internal wait-nodes 
that match up producers and consumers. It uses the synchronization state 
to allow a producer to proceed when a consumer takes the item, and vice-versa.

Users of the java.util.concurrent package may of course define their 
own synchronizers for custom applications. For example, among those 
that were considered but not adopted in the package are classes providing 
the semantics of various flavors of WIN32 events, binary latches, 
centrally managed locks, and tree-based barriers.

</pre>
</details>

下面是`java.util.concurrent`并发包下同步器类如何使用这个框架的概括:

`ReentrantLock`可重入锁类使用同步状态保存(递归)锁的数量. 当一个锁被获取, 它也会
记录当前线程的标识符来检查递归并检查非法状态异常, 当错误线程想要释放锁时.
这个类也使用提供的`ConditionObject`类, 并导出其他的监控和检测方法.
这个类支持一个可选的`公平`模式通过内部声明两个不同的`AbstractQueuedSynchronizer`子类
(公平锁禁止插队)并且在构造时设置每一个`ReentrantLock`实例使用合适的子类.

`ReentrantReadWriteLock`可重入读写锁类使用同步状态的16比特(右边)来保存写锁数量,
使用剩下的16比特(左边)保存读锁数量. 写锁的结构在其他方面和可重入锁一样. 读锁使用
`acquireShared`方法来允许多个读者.

`Semaphore`信号量类使用同步状态来代表数量. 当它到达0时, 所有获取`acquire`都可以通过.

`FutureTask`类使用同步状态代表`future`对象的运行状态(初始化, 运行, 取消, 完成).
调用`release`来设置或取消一个`future`, 通过`acquire`来解锁等待计算结果的线程.

`SynchronousQueue`同步队列类(一个CSP风格的变体)使用颞部的等待节点牌来匹配生产者
和消费者. 当消费者拿走物品时, 它使用同步状态允许生产者执行, 反之亦然.

`java.util.concurrent`并发包的使用者当然可以为自定义的应用开发他们自己的同步器.
例如, 被考虑过但没有被引入到这个包里的WIN32事件, 二进制门栓, 集中管理的锁, 基于树的帷幕.

以上是关于AQS框架的原理和使用, 论文最后还有关于性能指数的测量请参考原文.

[原文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)

## 致谢

*姬野永远提供翻译建议.*








