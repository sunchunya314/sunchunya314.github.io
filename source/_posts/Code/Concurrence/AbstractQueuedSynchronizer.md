---
title: AbstractQueuedSynchronizer
date: 2022-03-29 11:12:53
tags:
- 并发
- 多线程
categories:
- code
- concurrence
---

### 原理

#### 段1
 * Provides a framework for implementing blocking locks and related
 * synchronizers (semaphores, events, etc) that rely on
 * first-in-first-out (FIFO) wait queues.  This class is designed to
 * be a useful basis for most kinds of synchronizers that rely on a
 * single atomic <tt>int</tt> value to represent state. Subclasses
 * must define the protected methods that change this state, and which
 * define what that state means in terms of this object being acquired
 * or released. Given these, the other methods in this class carry
 * out all queuing and blocking mechanics. Subclasses can maintain
 * other state fields, but only the atomically updated <tt>int</tt>
 * value manipulated using methods {@link #getState}, {@link
 * #setState} and {@link #compareAndSetState} is tracked with respect
 * to synchronization.

>提供一个基于先进先出的等待队列的框架实现阻塞锁和相关的同步器(例如信号量, 事件等). 
>这个类仅仅依靠一个int的原子类来管理状态的设计作为大部分的同步器的基础.
>子类必须定义执行方法来改变这个状态, 并且定义当这个对象被`acquired`和`released`的时候对应的状态含义.
>考虑到这些, 该类中其他方法执行所有的排队阻塞机制.
>子类可以维护其他的状态字段, 但是只有使用方法`getState`,`setState`和`compareAndSetState`进行原子更新的`int`值是用来追踪同步的.

#### 段2

 * <p>Subclasses should be defined as non-public internal helper
 * classes that are used to implement the synchronization properties
 * of their enclosing class.  Class
 * <tt>AbstractQueuedSynchronizer</tt> does not implement any
 * synchronization interface.  Instead it defines methods such as
 * {@link #acquireInterruptibly} that can be invoked as
 * appropriate by concrete locks and related synchronizers to
 * implement their public methods.
 
> 子类应该被定义为非`public`的内部辅助类用来实现它们封装类的同步属性.
> `AbstractQueuedSynchronizer`类不实现任何同步接口.
> 它定义了`acquireInterruptibly`这样的方法被具体的锁视情况调用和相关的同步器来实现它们的公共方法.

 * <p>This class supports either or both a default <em>exclusive</em>
 * mode and a <em>shared</em> mode. When acquired in exclusive mode,
 * attempted acquires by other threads cannot succeed. Shared mode
 * acquires by multiple threads may (but need not) succeed. This class
 * does not &quot;understand&quot; these differences except in the
 * mechanical sense that when a shared mode acquire succeeds, the next
 * waiting thread (if one exists) must also determine whether it can
 * acquire as well. Threads waiting in the different modes share the
 * same FIFO queue. Usually, implementation subclasses support only
 * one of these modes, but both can come into play for example in a
 * {@link ReadWriteLock}. Subclasses that support only exclusive or
 * only shared modes need not define the methods supporting the unused mode.
 
> 这个类支持一个默认的互斥模式和一个共享模式.
> 当在互斥模式时被`acquire`, 其他`acquire`的线程不能成功.
> 共享模式下多线程同时`acquire`可能会(不是必须)成功.
> 这个类并不`理解`这些区别除了在这个机制当一个共享模式`acquire`成功, 另一个等待线程(如果存在)必须决定是否它也可以`acquire`.
> 在不同模式下等待的线程共享同一个`FIFO`队列. 
> 通常, 子类只实现支持这些模式中的一个, 但是可以通过`ReadWriteLock`来全部演示.
> 只支持互斥模式或者共享模式的子类不需要定义方法来支持没用到的模式.

# ConditionObject
 * <p>This class defines a nested {@link ConditionObject} class that
 * can be used as a {@link Condition} implementation by subclasses
 * supporting exclusive mode for which method {@link
 * #isHeldExclusively} reports whether synchronization is exclusively
 * held with respect to the current thread, method {@link #release}
 * invoked with the current {@link #getState} value fully releases
 * this object, and {@link #acquire}, given this saved state value,
 * eventually restores this object to its previous acquired state.  No
 * <tt>AbstractQueuedSynchronizer</tt> method otherwise creates such a
 * condition, so if this constraint cannot be met, do not use it.  The
 * behavior of {@link ConditionObject} depends of course on the
 * semantics of its synchronizer implementation.
 
> 这个类定义了一个嵌套类`ConditionObject`, 可以被子类用作支持互斥模式的`Condition`实现.
> 通过方法`isHeldExclusively`知道是否同步正在被当前线程互斥持有,被当前`getState`值调用的方法`release`完全的释放这个对象.
> 并且`acquire` 给予这个保存的`acquired`状态.
> 最终这个对象重新回到它先前的`acquired`状态.
> 没有一个`AbstractQueuedSynchronizer`方法创建一个`condition` 所以如果这个约束不能达成, 那么不用使用它. 
> `ConditionObject`的行为依赖于同步器的实现机制.

 * <p>This class provides inspection, instrumentation, and monitoring
 * methods for the internal queue, as well as similar methods for
 * condition objects. These can be exported as desired into classes
 * using an <tt>AbstractQueuedSynchronizer</tt> for their
 * synchronization mechanics.

> 这个类为内部队列和`conditon`对象提供观测, 构件, 监控器方法.
> 这些可以被导出到使用`AbstractQueuedSynchronizer`作为它们同步机制的类里.


 * <p>Serialization of this class stores only the underlying atomic
 * integer maintaining state, so deserialized objects have empty
 * thread queues. Typical subclasses requiring serializability will
 * define a <tt>readObject</tt> method that restores this to a known
 * initial state upon deserialization.
 
> 该类的序列化仅仅保存底层的原子`integer`维护状态, 所以解序列化对象只有空的线程队列. 
> 经典的子类需要序列化将定义一个`readObject`方法, 解序列化时用来重新加载自身数据到一个到初始化状态.


## 用法
 
 * <p>To use this class as the basis of a synchronizer, redefine the
 * following methods, as applicable, by inspecting and/or modifying
 * the synchronization state using {@link #getState}, {@link
 * #setState} and/or {@link #compareAndSetState}:
 
> 为了使用这个类作为一个同步器的基础, 重新定义以下方法, 以作可用的, 通过观测或修改同步状态
> `getState`
> `setState`
> `compareAndSetState`

 <ul>
 <li> tryAcquire
 <li> tryRelease
 <li> tryAcquireShared
 <li> tryReleaseShared
 <li> isHeldExclusively
 </ul>
 
  * Each of these methods by default throws {@link
  * UnsupportedOperationException}.  Implementations of these methods
  * must be internally thread-safe, and should in general be short and
  * not block. Defining these methods is the <em>only</em> supported
  * means of using this class. All other methods are declared
  * <tt>final</tt> because they cannot be independently varied.
  
> 以上这些方法中每个方法都默认会抛出异常`UnsupportedOperationException`.
> 这些方法的实现必须是线程安全的, 并且应该大体是简短, 非阻塞的.
> 定义这些方法是使用该类的唯一支持的方法.
> 所有其他的方法被定义为`final`因为它们不能被单独改变.

  * <p>You may also find the inherited methods from {@link
  * AbstractOwnableSynchronizer} useful to keep track of the thread
  * owning an exclusive synchronizer.  You are encouraged to use them
  * -- this enables monitoring and diagnostic tools to assist users in
  * determining which threads hold locks.
  *
  * <p>Even though this class is based on an internal FIFO queue, it
  * does not automatically enforce FIFO acquisition policies.  The core
  * of exclusive synchronization takes the form:

> 你可能一发现了从`AbstractOwnableSynchronizer`继承的方法有利于追踪拥有一个互斥同步器的线程.
> 你被鼓励使用它们.
> 这个使得监控器和诊断工具帮助用户决定哪一个线程获取锁.
>
> 即使该类是基于一个内部`FIFO`队列, 它并不会自动的强迫`FIFO`获取策略. 
> 互斥同步器的核心使用以下形式:

```textmate
  Acquire:
      while (!tryAcquire(arg)) {
           enqueue thread if it is not already queued;
           possibly block current thread;
      }
 
  Release:
      if (tryRelease(arg))
         unblock the first queued thread;
```
 * (Shared mode is similar but may involve cascading signals.)

> 共享模式类似, 但是可能包括大量的信号


 * <p><a name="barging">Because checks in acquire are invoked before
 * enqueuing, a newly acquiring thread may <em>barge</em> ahead of
 * others that are blocked and queued.  However, you can, if desired,
 * define <tt>tryAcquire</tt> and/or <tt>tryAcquireShared</tt> to
 * disable barging by internally invoking one or more of the inspection
 * methods, thereby providing a <em>fair</em> FIFO acquisition order.
 * In particular, most fair synchronizers can define <tt>tryAcquire</tt>
 * to return <tt>false</tt> if {@link #hasQueuedPredecessors} (a method
 * specifically designed to be used by fair synchronizers) returns
 * <tt>true</tt>.  Other variations are possible.
 
> 因为在`入队`之前`acquire`包含了检测, 一个新的正在`acquire`的线程可能会`插队`到其他正在排队或阻塞的线程前面.
> 然而, 你可以, 如果愿意的话定义`tryAcquire`并且/或者`tryAcquireShared`来禁止`插队`, 通过内部调用一个或则多个监测方法,由此提供一个公平的`FIFO`获取顺序.
> 实践中, 大部分公平的同步器可以定义`tryAcquire 来返回`false`, 如果`hasQueuedPredecessors`(为公平同步器专门设计的一个方法)返回`true`.
> 也可以是其他的变化.


 * <p>Throughput and scalability are generally highest for the
 * default barging (also known as <em>greedy</em>,
 * <em>renouncement</em>, and <em>convoy-avoidance</em>) strategy.
 * While this is not guaranteed to be fair or starvation-free, earlier
 * queued threads are allowed to recontend before later queued
 * threads, and each recontention has an unbiased chance to succeed
 * against incoming threads.  Also, while acquires do not
 * &quot;spin&quot; in the usual sense, they may perform multiple
 * invocations of <tt>tryAcquire</tt> interspersed with other
 * computations before blocking.  This gives most of the benefits of
 * spins when exclusive synchronization is only briefly held, without
 * most of the liabilities when it isn't. If so desired, you can
 * augment this by preceding calls to acquire methods with
 * "fast-path" checks, possibly prechecking {@link #hasContended}
 * and/or {@link #hasQueuedThreads} to only do so if the synchronizer
 * is likely not to be contended.
 
> 吞吐量和可观测行对于默认的冲突(也被称为贪婪的放弃和车队避让)策略来说是最重要的.
> 但是这不能保证公平或免于饥饿, 先入队的线程允许和后入队的线程竞争, 并且每次竞争有一个无差别的机会获得成功.
> 并且, 当`acquire`没有在通常意义上的自旋, 在阻塞之前它们可能会多次调用`tryAcquire`并掺杂着其他的计算.
> 当互斥同步纸只是唯一持有时, 这个给自旋带来了极大的优势, 不是这个情况就会失去很多优势.
> 如果需要的话, 你可以增加这个通过先前的调用来请求方法`最快路径`检查, 使用预检查的方法`hasContended`和/或`hasQueuedThreads`可以使得同步器减少竞争的情况.

 * <p>This class provides an efficient and scalable basis for
 * synchronization in part by specializing its range of use to
 * synchronizers that can rely on <tt>int</tt> state, acquire, and
 * release parameters, and an internal FIFO wait queue. When this does
 * not suffice, you can build synchronizers from a lower level using
 * {@link java.util.concurrent.atomic atomic} classes, your own custom
 * {@link java.util.Queue} classes, and {@link LockSupport} blocking
 * support.
 
> 这个类为同步器提供一个高效的, 可观测的基准的方式一部分是依靠以下参数
> 1.`int` state
> 2.`acquire` 
> 3.`release`
> 4.一个`FIFO`等待队列
> 如果这些还不满足需要, 你可以在一个低级别上自己构建同步器, 使用如下元素
> 1.`java.util.concurrent.atomic`包下原子类
> 2.`java.util.Queue`队列类
> 3.`LockSupport`阻塞支持


### 使用举例

 * <h3>Usage Examples</h3>
 * <p>Here is a non-reentrant mutual exclusion lock class that uses
 * the value zero to represent the unlocked state, and one to
 * represent the locked state. While a non-reentrant lock
 * does not strictly require recording of the current owner
 * thread, this class does so anyway to make usage easier to monitor.
 * It also supports conditions and exposes
 * one of the instrumentation methods:
 
> 下面是一个不可重入的互斥锁类, 使用`0`代表未锁定状态, `1`表示锁定状态.
> 因为一个非可重入的锁没有严格要求记录当前拥有线程, 这样实现可以让`monitor`用起来更简单.
> 它也支持`condition`和暴露其中的构件方法:

```java
  class Mutex implements Lock, java.io.Serializable {
 
    // 我们的内部帮助类
    private static class Sync extends AbstractQueuedSynchronizer {
      // 报告是否在锁的状态
      protected boolean isHeldExclusively() {
        return getState() == 1;
      }
 
      // 获取锁如果state是0
      public boolean tryAcquire(int acquires) {
        assert acquires == 1; // Otherwise unused
        if (compareAndSetState(0, 1)) {
          setExclusiveOwnerThread(Thread.currentThread());
          return true;
        }
        return false;
      }
 
      // 释放锁: 设置state为0
      protected boolean tryRelease(int releases) {
        assert releases == 1; // Otherwise unused
        if (getState() == 0) throw new IllegalMonitorStateException();
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
      }
 
      // 提供condition
      Condition newCondition() { return new ConditionObject(); }
 
      // 解序列化
      private void readObject(ObjectInputStream s)
          throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
      }
    }
 
    //  sync对象实现最复杂的工作
    private final Sync sync = new Sync();
 
    public void lock()                { sync.acquire(1); }
    public boolean tryLock()          { return sync.tryAcquire(1); }
    public void unlock()              { sync.release(1); }
    public Condition newCondition()   { return sync.newCondition(); }
    public boolean isLocked()         { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException {
      sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
      return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
  }
```

 * <p>Here is a latch class that is like a {@link CountDownLatch}
 * except that it only requires a single <tt>signal</tt> to
 * fire. Because a latch is non-exclusive, it uses the <tt>shared</tt>
 * acquire and release methods.
 
> 下面是一个类似`CountDownLatch`的门栓类
> 它只`acquire`一个单一的`signal`来唤醒.
> 因为一个门栓是非互斥的, 它使用`shared`来`acquire`和`release`方法.


```java
class BooleanLatch {
    // 内部帮助类
    private static class Sync extends AbstractQueuedSynchronizer {
      // 是否唤醒 state: 非0唤醒, 0未唤醒
      boolean isSignalled() { return getState() != 0; }
      // 唤醒 1, 未唤醒 -1, 返回值<0, 表示可以获取
      protected int tryAcquireShared(int ignore) {
        return isSignalled() ? 1 : -1;
      }
      // 将state设为1 来释放
      protected boolean tryReleaseShared(int ignore) {
        setState(1);
        return true;
      }
    }
 
    private final Sync sync = new Sync();
    public boolean isSignalled() { return sync.isSignalled(); }
    // 唤醒其他线程
    public void signal()         { sync.releaseShared(1); }
    // 当前线程等待
    public void await() throws InterruptedException { sync.acquireSharedInterruptibly(1); }
}

```

When meaningful, each synchronizer supports:
* Nonblocking synchronization attempts (for example,
tryLock) as well as blocking versions.
* Optional timeouts, so applications can give up waiting.
* Cancellability via interruption, usually separated into one
version of acquire that is cancellable, and one that isn't

同步器应该支持:
* 非阻塞的同步尝试(比如, `tryLock`) 和阻塞版本.
* 可选的超时, 这样应用可以放弃等待.
* 通过中断来取消, 通常是分开在两个版本的`acquire`,一个是可取消, 一个不能.


JVM优化锁的策略是0竞争.


```groovy

    while (!isOnSyncQueue(node)) {
        // park
        LockSupport.park(this);
        // todo 循环里的park 后面一定跟着中断检查
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
```



https://sunchunya314.github.io/
