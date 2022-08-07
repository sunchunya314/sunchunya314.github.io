---
title: BlockingQueue
date: 2022-06-19 10:08:51
tags:
---

### 原理

原文
A Queue that additionally supports operations that wait for the queue to become non-empty 
when retrieving an element, and wait for space to become available in the queue when 
storing an element.

译文
> 一个队列, 添加额外功能:
> 1. 当获取一个元素,等待队列非空再返回
> 2. 当存储一个元素时, 等待队列有可用空间再保存

原文
BlockingQueue methods come in four forms, with different ways of handling operations 
that cannot be satisfied immediately, but may be satisfied at some point in the 
future: one throws an exception, the second returns a special value 
(either null or false, depending on the operation), the third blocks 
the current thread indefinitely until the operation can succeed, and the fourth blocks 
for only a given maximum time limit before giving up. These methods are summarized 
in the following table:

译文
> `BlockingQueue` 方法有四种形式,带有不同方式的处理那些不能立刻响应但可能在将来某个时间响应的操作
> 1. 抛异常
> 2. 返回一个特殊值(`null`或`false`由操作决定)
> 3. 永久阻塞当前线程, 直到操作成功
> 4. 阻塞有一个最大时间限制, 超过就放弃.
如下表所示:
 <table class="plain">
 <caption>Summary of BlockingQueue methods</caption>
  <tr>
    <td></td>
    <th scope="col" style="font-weight:normal; font-style:italic">Throws exception</th>
    <th scope="col" style="font-weight:normal; font-style:italic">Special value</th>
    <th scope="col" style="font-weight:normal; font-style:italic">Blocks</th>
    <th scope="col" style="font-weight:normal; font-style:italic">Times out</th>
  </tr>
  <tr>
    <th scope="row" style="text-align:left">Insert</th>
    <td>add(Obj)</td>
    <td>offer(Obj) </td>
    <td>put(Obj) </td>
    <td>offer(Obj, long, TimeUnit)</td>
  </tr>
  <tr>
    <th scope="row" style="text-align:left">Remove</th>
    <td>remove() </td>
    <td>poll()</td>
    <td>take()</td>
    <td>poll(long, TimeUnit)</td>
  </tr>
  <tr>
    <th scope="row" style="text-align:left">Examine</th>
    <td>element()</td>
    <td>peek()</td>
    <td style="font-style: italic">not applicable</td>
    <td style="font-style: italic">not applicable</td>
  </tr>
 </table>

原文
A BlockingQueue does not accept null elements. Implementations throw 
NullPointerException on attempts to add, put or offer a null. A null 
is used as a sentinel value to indicate failure of poll operations.
A BlockingQueue may be capacity bounded. At any given time it may have 
a remainingCapacity beyond which no additional elements can be put without 
blocking. A BlockingQueue without any intrinsic capacity constraints always reports
a remaining capacity of Integer.MAX_VALUE.

译文
> 一个阻塞队列不接受 `null` 元素. 如果执行添加, 更新或者提供一个`null`值将会抛出空指针. 
> 一个`null`值是用来暗示取出操作失败的哨兵值. 一个阻塞队列可能有容量限制. 在任何给定的时间，
> 它可能有一个剩余容量，超过这个容量，没有额外的元素可以不阻塞。一个阻塞队列没有任何内嵌的约束容量
> , 剩余容量总是返回 `Integer.MAX_VALUE`

原文
BlockingQueue implementations are designed to be used primarily for 
producer-consumer queues, but additionally support the Collection interface. 
So, for example, it is possible to remove an arbitrary element from a queue 
using remove(x). However, such operations are in general not performed very 
efficiently, and are intended for only occasional use, such as when a 
queued message is cancelled.

译文
> 阻塞队列实现主要是设计用作 `生产者-消费者` 队列, 但是额外的支持集合接口.
> 所以, 举例来说, 它可以使用 `remove(x)` 从队列里武断的删除一个元素. 
> 然而, 这样的操作总的来说效率不是很高, 只有当一个队列消息被取消时偶尔使用.

原文
BlockingQueue implementations are thread-safe. All queuing methods achieve 
their effects atomically using internal locks or other forms of concurrency 
control. However, the bulk Collection operations addAll, containsAll, 
retainAll and removeAll are not necessarily performed atomically unless 
specified otherwise in an implementation. So it is possible, for example, 
for addAll(c) to fail (throwing an exception) after adding only some of the 
elements in c.

译文
> 阻塞队列的实现是线程安全的. 所有的队列方法使用内置锁或其它形式的并发控制来原子性地达到它们效果.
> 然而, 整体的集合操作 `addAll`, `containsAll`, `retainAll` 和 `removeAll` 没有必要实现原子
> 操作, 除非在实现里特殊处理. 举例, 所以对于 `addAll(c)`有可能失败(抛出异常)在添加了集合 `c` 部分的
> 元素之后.

原文
A BlockingQueue does not intrinsically support any kind of "close" or "shutdown" 
operation to indicate that no more items will be added. The needs and usage of 
such features tend to be implementation-dependent. For example, a common tactic 
is for producers to insert special end-of-stream or poison objects, that are 
interpreted accordingly when taken by consumers.

译文
> 一个阻塞队列没有内置的支持任何形式的`close` 或者 `shutdown` 操作来暗示不允许添加元素. 
> 这个功能的需要和使用趋向于独立实现. 例如, 一个常见的手段给生产插入一个特殊的`结束流`或者`毒对象`,
> 那么当别消费者拿到时会因此中断.

原文
Usage example, based on a typical producer-consumer scenario. Note that a 
BlockingQueue can safely be used with multiple producers and multiple consumers.

<p>Memory consistency effects: As with other concurrent
collections, actions in a thread prior to placing an object into a
{@code BlockingQueue}
<a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a>
actions subsequent to the access or removal of that element from
the {@code BlockingQueue} in another thread.

<p>This interface is a member of the
<a href="{@docRoot}/../technotes/guides/collections/index.html">
Java Collections Framework</a>.

```java
  class Producer implements Runnable {
    private final BlockingQueue queue;
    Producer(BlockingQueue q) { queue = q; }
    public void run() {
      try {
        while (true) { queue.put(produce()); }
      } catch (InterruptedException ex) { //... handle ...}
    }
    Object produce() { //... }
  }
 
  class Consumer implements Runnable {
    private final BlockingQueue queue;
    Consumer(BlockingQueue q) { queue = q; }
    public void run() {
      try {
        while (true) { consume(queue.take()); }
      } catch (InterruptedException ex) { //... handle ...}
    }
    void consume(Object x) { //... }
  }
 
  class Setup {
    void main() {
      BlockingQueue q = new SomeQueueImplementation();
      Producer p = new Producer(q);
      Consumer c1 = new Consumer(q);
      Consumer c2 = new Consumer(q);
      new Thread(p).start();
      new Thread(c1).start();
      new Thread(c2).start();
    }
  }
```

译文
> 使用案例, 基于一个经典的生产者-消费者场景. 注意到一个阻塞队列可以被多个生产者和多个消费者安全的使用.
> 内存一致性影响: 一个线程将对象放入 `BlockQueue`的行为 `happen-before` 另一个线程访问或删除该
> 元素的行为.
> 这个接口属于Java集合的成员.


### API

```java
    /**
     * Inserts the specified element into this queue if it is possible to do
     * so immediately without violating capacity restrictions, returning
     * {@code true} upon success and throwing an
     * {@code IllegalStateException} if no space is currently available.
     * When using a capacity-restricted queue, it is generally preferable to
     * use {@link #offer(Object) offer}.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Collection#add})
     * @throws IllegalStateException if the element cannot be added at this
     *         time due to capacity restrictions
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null
     * @throws IllegalArgumentException if some property of the specified
     *         element prevents it from being added to this queue
     */
    boolean add(E e);
```

译文
```java
    boolean add(E e);
```
* 插入特定的元素到队列里, 如果可以这么做, 那么不需要违反容量限制, 一旦成功立刻返回`true`
* 如果当前没有可用空间会报出一个`IllegalStateException`异常.
* 当使用一个`容量限制`队列时,需要失败返回false, 使用`offer`方法, .
* `e` 添加的元素
* `return true` 同 `Collection#add`
* `IllegalStateException` 如果元素此刻由于容量限制不能被添加
* `ClassCastException` 如果元素的类阻止它被加入到这个队列
* `NullPointerException` 如果指定元素是`null`
* `IllegalArgumentException` 如果元素的某个属性阻止它被加入到这个队列



原文
```java
    /**
     * Inserts the specified element into this queue if it is possible to do
     * so immediately without violating capacity restrictions, returning
     * {@code true} upon success and {@code false} if no space is currently
     * available.  When using a capacity-restricted queue, this method is
     * generally preferable to {@link #add}, which can fail to insert an
     * element only by throwing an exception.
     *
     * @param e the element to add
     * @return {@code true} if the element was added to this queue, else
     *         {@code false}
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null
     * @throws IllegalArgumentException if some property of the specified
     *         element prevents it from being added to this queue
     */
    boolean offer(E e);
```

译文
```java
    boolean offer(E e);
```
* 插入特定的元素到队列里, 如果可以这么做, 那么不需要违反容量限制, 一旦成功立刻返回`true`
* 如果当前没有可用空间, 会返回`false`.
* 当使用一个`容量限制`队列时,需要失败抛出异常, 使用`add`方法.
* `e` 添加的元素
* `return true` 当元素被加入到这个队列, 否则返回`false`
* `ClassCastException` 如果元素的类阻止它被加入到这个队列
* `NullPointerException` 如果指定元素是`null`
* `IllegalArgumentException` 如果元素的某个属性阻止它被加入到这个队列


原文
```java
    /**
     * Inserts the specified element into this queue, waiting if necessary
     * for space to become available.
     *
     * @param e the element to add
     * @throws InterruptedException if interrupted while waiting
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null
     * @throws IllegalArgumentException if some property of the specified
     *         element prevents it from being added to this queue
     */
    void put(E e) throws InterruptedException;
```

译文
```java
    void put(E e) throws InterruptedException;
```
* 插入元素到队列, 会一直等待直到有可用空间
* `e` 被添加的元素
* `InterruptedException` 等待时被中断
* `ClassCastException` 如果元素的类阻止它被加入到这个队列
* `NullPointerException` 如果指定元素是`null`
* `IllegalArgumentException` 如果元素的某个属性阻止它被加入到这个队列

原文
```java
    /**
     * Inserts the specified element into this queue, waiting up to the
     * specified wait time if necessary for space to become available.
     *
     * @param e the element to add
     * @param timeout how long to wait before giving up, in units of
     *        {@code unit}
     * @param unit a {@code TimeUnit} determining how to interpret the
     *        {@code timeout} parameter
     * @return {@code true} if successful, or {@code false} if
     *         the specified waiting time elapses before space is available
     * @throws InterruptedException if interrupted while waiting
     * @throws ClassCastException if the class of the specified element
     *         prevents it from being added to this queue
     * @throws NullPointerException if the specified element is null
     * @throws IllegalArgumentException if some property of the specified
     *         element prevents it from being added to this queue
     */
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
```
译文
```java
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
```

* 插入元素到队列里, 如果需要使空间可用, 等待指定的等待时间.
* `e` 被添加的元素
* `timeout` 等待多久才放弃, 以参数`unit`为单位
* `unit` 一个`TimeUnit`决定如何中断`timeout`参数
* 如果成功, 返回`true`, 或者在空间可用之前指定等待时间超时, 返回`false`
* `InterruptedException` 等待时被中断
* `ClassCastException` 如果元素的类阻止它被加入到这个队列
* `NullPointerException` 如果指定元素是`null`
* `IllegalArgumentException` 如果元素的某个属性阻止它被加入到这个队列

原文
```java
    /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element becomes available.
     *
     * @return the head of this queue
     * @throws InterruptedException if interrupted while waiting
     */
    E take() throws InterruptedException;
```

译文
```java
    E take() throws InterruptedException;
```
* 获取并移除队列的头一个元素, 保持等待直到有可用元素.
* 返回队列头.
* `InterruptedException` 如果等待时发生中断.

原文
```java
    /**
     * Retrieves and removes the head of this queue, waiting up to the
     * specified wait time if necessary for an element to become available.
     *
     * @param timeout how long to wait before giving up, in units of
     *        {@code unit}
     * @param unit a {@code TimeUnit} determining how to interpret the
     *        {@code timeout} parameter
     * @return the head of this queue, or {@code null} if the
     *         specified waiting time elapses before an element is available
     * @throws InterruptedException if interrupted while waiting
     */
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;
```

译文
```java
  E poll(long timeout, TimeUnit unit)
        throws InterruptedException;
```

* 获取并移除队列的头一个元素, 等待指定等待时间直到元素可用
* `timeout` 等待多久才放弃, 以参数`unit`为单位
* `unit` 一个`TimeUnit`决定如何中断`timeout`参数
* 返回队列头, 或者等待元素可用时过了指定的等待时间则返回`null`
* `InterruptedException` 如果等待时发生中断.

原文
```java
   /**
     * Returns the number of additional elements that this queue can ideally
     * (in the absence of memory or resource constraints) accept without
     * blocking, or {@code Integer.MAX_VALUE} if there is no intrinsic
     * limit.
     *
     * <p>Note that you <em>cannot</em> always tell if an attempt to insert
     * an element will succeed by inspecting {@code remainingCapacity}
     * because it may be the case that another thread is about to
     * insert or remove an element.
     *
     * @return the remaining capacity
     */
    int remainingCapacity();
```

译文

```java
    int remainingCapacity();
```
* 返回这个队列理论上不需要阻塞可以接收的其它的元素的数量, 如果没有约束限制, 会返回`Integer.MAX_VALUE`
* 注意: 你不能通过监测`remainingCapacity`的值来辨别一个插入元素的尝试是否会成功, 因为它可以会受到另一个线程插入或删除一个元素的影响

原文
```java
    /**
     * Removes a single instance of the specified element from this queue,
     * if it is present.  More formally, removes an element {@code e} such
     * that {@code o.equals(e)}, if this queue contains one or more such
     * elements.
     * Returns {@code true} if this queue contained the specified element
     * (or equivalently, if this queue changed as a result of the call).
     *
     * @param o element to be removed from this queue, if present
     * @return {@code true} if this queue changed as a result of the call
     * @throws ClassCastException if the class of the specified element
     *         is incompatible with this queue
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     * @throws NullPointerException if the specified element is null
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     */
    boolean remove(Object o);
```

译文
```java
    boolean remove(Object o);
```
* 从队列中删除一个指定元素的实例, 如果存在的话.
* 更正式的说, 根据`o.equals(e)`来删除一个元素, 如果这个队列包含一个或多个这样的元素.
* 返回`true`, 如果这个队列包含这个特定的元素(或者等价地, 如果这个队列因为这个调用改变了)
* `o`: 从这个队列移除的元素, 如果存在的话.
* `ClassCastException`: 如果这个指定元素的类和这个队列不兼容.
* `NullPointerException`: 如果指定元素是`null`


原文
```java
    /**
     * Returns {@code true} if this queue contains the specified element.
     * More formally, returns {@code true} if and only if this queue contains
     * at least one element {@code e} such that {@code o.equals(e)}.
     *
     * @param o object to be checked for containment in this queue
     * @return {@code true} if this queue contains the specified element
     * @throws ClassCastException if the class of the specified element
     *         is incompatible with this queue
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     * @throws NullPointerException if the specified element is null
     *         (<a href="../Collection.html#optional-restrictions">optional</a>)
     */
    public boolean contains(Object o);
```

译文
```java
  public boolean contains(Object o);
```

* 返回`true`如果这个队列包含这个特定的元素.
* 更正式的说, 当且仅当这个队列根据`o.equals(e)`判断包含至少一个元素的时候返回`true`
* `o`: 被用来检查包含在这个队列的对象
* `ClassCastException`: 如果这个指定元素的类和这个队列不兼容.
* `NullPointerException`: 如果指定元素是`null`

原文
```java
    /**
     * Removes all available elements from this queue and adds them
     * to the given collection.  This operation may be more
     * efficient than repeatedly polling this queue.  A failure
     * encountered while attempting to add elements to
     * collection {@code c} may result in elements being in neither,
     * either or both collections when the associated exception is
     * thrown.  Attempts to drain a queue to itself result in
     * {@code IllegalArgumentException}. Further, the behavior of
     * this operation is undefined if the specified collection is
     * modified while the operation is in progress.
     *
     * @param c the collection to transfer elements into
     * @return the number of elements transferred
     * @throws UnsupportedOperationException if addition of elements
     *         is not supported by the specified collection
     * @throws ClassCastException if the class of an element of this queue
     *         prevents it from being added to the specified collection
     * @throws NullPointerException if the specified collection is null
     * @throws IllegalArgumentException if the specified collection is this
     *         queue, or some property of an element of this queue prevents
     *         it from being added to the specified collection
     */
    int drainTo(Collection<? super E> c);
```

译文
```java
    int drainTo(Collection<? super E> c);
```

* 从这个队列里删除所有可访问的元素, 并把它们添加到指定的集合中
* 这个操作可能回避重复从队列里`poll`元素效率更高.
* 当尝试往集合`c`中添加元素由于关联的异常被抛出而失败时可能会导致元素不在任意一个集合中.
* 当试图清空一个队列到它自身时, 会抛出`IllegalArgumentException`异常.
* 进一步地, 当`drainTo`这个操作在进行时, 如果指定集合被修改了, 这个操作的行为未定义.
* `c`: 把元素转移到这个集合
* 返回被转移的元素的数量
* `UnsupportedOperationException`: 如果指定集合不支持添加这些元素
* `ClassCastException`: 如果这个队列元素的类组织它被添加到指定集合.
* `NullPointerException`: 如果指定集合是`null`
* `IllegalArgumentException`: 如果指定集合是这个队列本身, 或者这个队列的一个元素的属性阻止它加入到指定集合.

原文
```java
    /**
     * Removes at most the given number of available elements from
     * this queue and adds them to the given collection.  A failure
     * encountered while attempting to add elements to
     * collection {@code c} may result in elements being in neither,
     * either or both collections when the associated exception is
     * thrown.  Attempts to drain a queue to itself result in
     * {@code IllegalArgumentException}. Further, the behavior of
     * this operation is undefined if the specified collection is
     * modified while the operation is in progress.
     *
     * @param c the collection to transfer elements into
     * @param maxElements the maximum number of elements to transfer
     * @return the number of elements transferred
     * @throws UnsupportedOperationException if addition of elements
     *         is not supported by the specified collection
     * @throws ClassCastException if the class of an element of this queue
     *         prevents it from being added to the specified collection
     * @throws NullPointerException if the specified collection is null
     * @throws IllegalArgumentException if the specified collection is this
     *         queue, or some property of an element of this queue prevents
     *         it from being added to the specified collection
     */
    int drainTo(Collection<? super E> c, int maxElements);
```

译文
```java
    int drainTo(Collection<? super E> c, int maxElements);
```

* 从这个队列里删除最多给定数量的可访问的元素, 并把它们添加到指定的集合中
* 当尝试往集合`c`中添加元素由于关联的异常被抛出而失败时可能会导致元素不在任意一个集合中.
* 当试图清空一个队列到它自身时, 会抛出`IllegalArgumentException`异常.
* 进一步地, 当`drainTo`这个操作在进行时, 如果指定集合被修改了, 这个操作的行为未定义.
* `c`: 把元素转移到这个集合
* `maxElements`: 转移元素的最大数量
* 返回被转移的元素的数量
* `UnsupportedOperationException`: 如果指定集合不支持添加这些元素
* `ClassCastException`: 如果这个队列元素的类组织它被添加到指定集合.
* `NullPointerException`: 如果指定集合是`null`
* `IllegalArgumentException`: 如果指定集合是这个队列本身, 或者这个队列的一个元素的属性阻止它加入到指定集合.












