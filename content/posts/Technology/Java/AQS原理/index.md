---
title: 从ReentrantLock看AQS原理
date: 2021-05-13 15:33:07
tags:
---
## ReentrantLock的使用

```java
ReentrantLock lock = new ReentrantLock();
try {
    lock.lock();
} finally {
    lock.unlock();
}
```

## AQS原理概述

```java
private volatile int state;
```

维护了一个`int state`作为状态机，CLH队列`Node`暂存阻塞的线程，通过判断状态以及CAS变更状态机，遵循**FIFO**方式从队列中拿出线程运行。

## java中内置的实现：

- ReentrantLock(可重入锁)
- Semaphore（信号量）
- CountDownLatch（计数锁，这个是老朋友了）
- ReentrantReadWriteLock可重入读写锁
- SynchronousQueue
- FutureTask
- ThreadPoolExecutor中的work

## AQS锁分类

- 独占锁：也叫排他锁，即**锁只能由一个线程获取**，若一个线程获取了锁，则**其他想要获取锁的线程只能等待，直到锁被释放**。eg:写锁；
- 共享锁：**锁可以由多个线程同时获取，锁被获取一次，则锁的计数器+1**。eg:读锁；

## 核心方法

### 抽象方法：

- `tryAcquire` 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态。
- `tryRelease` 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态；
- `tryAcquireShared` 共享式获取同步状态，返回>=0的值，表示获取成功，反之获取失败
- `tryReleaseShared` 共享式释放同步状态
- `isHeldExclusively` 当前同步器是否再独占模式下被线程占用，一般该方法表示是否被当前线程所独占

>记忆方法：只记`Acquire`和`release`，前面加shared就是共享，没有就是独占。

### 模板方法：

- `acquire` 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用重写的`tryAcquire(int arg)`方法。
- `acquireInterruptibly` 与上面方法作用相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException并返回
- `tryAcquireNanos(int arg,Long nanos)` 在上个方法基础上增加了超时限制，如果当前线程在超时时间内没有获取到同步状态，那么将会返回false，如果获取到了返回true
- `acquireShared(int arg)` 共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同意时刻可以有多个线程获取到同步状态
- `acquireSharedInterruptibly(int arg)` 与上个方法相同，但是会响应中断
- `tryAcquireSharedNanos` 在上个方法基础上怎加了超时机制
- `release（int arg)` 独占式的释放同步状态，该方法会在释放同步状态后，将同步队列中第一个节点包含的线程唤醒
- `releaseShared(int arg)` 共享式释放同步状态


## 同步队列CLH

同步队列是一个遵循FIFO（first in first out）原则的双向链表。双向链包含前驱节点和后继节点。通常用于自旋同步场景。

入队时需要创建一个新的节点拼接到队尾，而出队时只需要把队首置空就行了。

### AQS中的同步队列

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    //队首引用
    private transient volatile Node head;
    //队尾引用
    private transient volatile Node tail;

    static final class Node {
        // 表示线程取消申请锁
        static final int CANCELLED =  1;
    
        // 表示线程正在申请锁，等待被分配
        static final int SIGNAL    = -1;
        
        // 表示线程在等待某些条件达成，再进入下一阶段
        static final int CONDITION = -2;
        
        // 表示把对当前节点进行的操作，继续往队列传播下去
        static final int PROPAGATE = -3;
        
        // 表示当前线程的状态，有五种
        volatile int waitStatus;

        // 双向链之一：前驱节点
        volatile Node prev;
        // 双向链之一：后继节点
        volatile Node next;

        // 节点代表的线程
        volatile Thread thread;

        // 条件队列——单向链，链接下一个节点
        Node nextWaiter;
    }
```

![](/images/CLH1.jpg)

AQS通过这个同步队列来维护当前获取锁失败处于阻塞状态的线程。

在这个队列中，每一个node对应一个线程，当一个线程获取锁失败，会被加入到队尾。只有链首是处于运行状态的线程，其他都在阻塞中。当队首的线程释放锁时，会唤醒下一个node，而下一个node中的线程会尝试获取锁。如果成功，则将上一个node移除，使自己至于链首。

### Node的五种状态

- 0：新结点入队时的默认状态；
- CANCELLED(1)：表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
- SIGNAL(-1)：表示后继结点在等待当前结点唤醒。后继结点入队时，会将当前结点的状态更新为SIGNAL。
- CONDITION(-2)：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
- PROPAGATE(-3)：共享模式下，前驱结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。

### Node的双向链结构

以Node的结构来看，prev 和 next 属性将可以支持AQS可以将请求锁的线程构成双向队列，而入队列出队列，以及先入先出的特性，需要方法来支持。

```java
private transient volatile Node head;
private transient volatile Node tail;

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { 
            // 进入到这里，说明没有head节点，CAS操作创建一个head节点
            if (compareAndSetHead(new Node()))
                // 如果这里失败，说明发生了并发，会再次循环并走到下面的else
                tail = head;
        } else {
            node.prev = t;
            // 把Node加入到尾部，保证加入到为止，并发会重走
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

AQS中，以head为CLH队列头部，以tail为CLH队列尾部，当加入节点时，通过CAS和自旋保证节点正确入队（加入队尾）。

## ReentrantLock原理

### 构造方法

```java
//无参构造方法
public ReentrantLock() {
    sync = new NonfairSync();
}
//有参构造方法
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

构造方法中只是初始化了一个`Sync`，这就是个AQS：

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
```

那么构造方法的作用就是初始化`Sync`这个AQS，但是根据参数，有两种实现方式：

- FairSync公平锁：也叫独占锁，多个线程按照申请锁的顺序去获得锁，后申请锁的线程需要排队，等它之前的线程获得锁并释放后，它才能获得锁；
- NonfairSync非公平锁：线程获得锁的顺序于申请锁的顺序无关，申请锁的线程可以直接尝试获得锁，谁抢到就是谁的。

### lock()

```java
//注意这里传递了1
public void lock() {
    sync.acquire(1);
}
```

**注意，lock时固定传了整数`1`。**

加锁，就是调用了AQS的`acquire(int arg)`方法。内部实现是：

```java
 public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //中断当前线程
        selfInterrupt();
}
```

1. `tryAcquire(arg)`尝试获取锁；
2. 如果拿不到，则`addWaiter(Node.EXCLUSIVE)`创建一个等待节点；
3. 然后`acquireQueued(waiter)`添加进等待队列；
4. 最后中断。
#### tryAxquire(int arg)尝试获取锁

这个方法的作用是**尝试获取锁，返回Boolean表示是否成功获取锁**，这是个抽象方法，他在公平锁和非公平锁的实现有差别。

>当我看完部分源码后，不得不佩服老外的命名能力，try用的非常准确，真的只是尝试获取锁，并不保证一定能获取到锁。

##### 公平锁

```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        //尝试获取锁
        @ReservedStackAccess
        protected final boolean tryAcquire(int acquires) {
            //拿到当前线程
            final Thread current = Thread.currentThread();
            //拿到AQS的状态
            int c = getState();
            if (c == 0) {
                //如果AQS是初始化状态
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                        //如果没有线程正在等待，并且CAS由0到1通过，把当前线程缓存为独占线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                //如果当前线程就是独占线程
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //就重置AQS状态计数
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    //查询是否有线程正在等待
    public final boolean hasQueuedPredecessors() {
        Node h, s;
        if ((h = head) != null) {
            //如果head不为空，
            if ((s = h.next) == null || s.waitStatus > 0) {
                //如果head的next不为空，或者head处于等待状态
                s = null; 
                for (Node p = tail; p != h && p != null; p = p.prev) {
                    //从队尾遍历到队首
                    if (p.waitStatus <= 0)
                    //找到最早开始等待的node
                        s = p;
                }
            }
            if (s != null && s.thread != Thread.currentThread())
                //最后找到[最早开始等待的node]并且它所在的线程不是当前线程，如果有就返回true
                return true;
        }
        return false;
    }
```

在`tryAcquire`方法中，首先判断AQS的state是否为0，为0表示当前没有线程持有锁，在锁待命的状态下才会去获取锁。

然后判断是否有其他线程在等待锁，没有的话就CAS那个state由0到1（防止其他线程竞争锁，先到先得），通过就把当前线程设置为AQS的独占线程，返回true，这样就使得线程申请到了锁。

如果不是初始化状态，判断当前线程是否独占线程，是的话直接返回true，并且把state加1并返回true。这里充分体现了**可重入锁**的特点。

那么如果有线程正在等待锁会怎样呢？当前线程会获取锁失败，会返回false，然后AQS会把当前线程封装成Node，添加到CLH的队尾，后面会详细说明。

`hasQueuedPredecessors`方法负责从队尾遍历到队首，找到最早开始等待并且持有其他线程的node，如果存在，就说明有线程处于等待状态。

总结一下可重入锁的原理：

1. 通过CAS改变state并且判断state状态，避免锁竞争；
2. 已经持有锁的线程可免除CAS再次获取锁。

再总结一下公平锁的原理：**判断当前锁是否被占用，再遍历CLH队列，如果存在处于等待状态的其他线程，则当前线获取锁失败，改变为等待状态，添加到CLH的队尾，以保证每个线程按序持有锁。**

##### 非公平锁

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        protected final boolean tryAcquire(int acquires) {
            //调用父类的方法
            return nonfairTryAcquire(acquires);
        }
    }
```

nonfairTryAcquire方法位于父类`Sync`中，那么这里也体现了模板方法模式。

```java
final boolean nonfairTryAcquire(int acquires) {
    //拿到当前线程
    final Thread current = Thread.currentThread();
    //拿到AQS中的state
    int c = getState();
    if (c == 0) {
        //如果是初始状态
        if (compareAndSetState(0, acquires)) {
            //如果CAS由0到1通过，那么就把当前线程设置为外部线程，到这里
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

从代码来看，非公平锁并没有判断是否有线程正在等待锁，所以当**新线程申请锁时，能拿到锁就拿，拿不到再加入队列去等。**

##### ReentrantLock的公平与非公平锁都是可重入的

##### 公平与非公平锁的区别

公平锁：多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队首的线程才能得到锁。

- 优点：所有的线程都能得到资源，不会饿死在队列中。
- 缺点：吞吐量会下降很多，队列里面除了第一个线程，其他的线程都会阻塞，cpu唤醒阻塞线程的开销会很大。

非公平锁：多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。

- 优点：可以减少CPU唤醒线程的开销，CPU也不必取唤醒所有线程，会减少唤起线程的数量。
- 缺点：你们可能也发现了，这样有可能导致队列中的线程一直获取不到锁或者长时间获取不到锁，导致饿死。

#### addWaiter()

![](/images/CLH2.jpg)

我们在分析其入列操作在简单不过。无非就是将tail（使用CAS保证原子操作）指向新节点，新节点的prev指向队列中最后一节点（旧的tail节点），原队列中最后一节点的next节点指向新节点以此来建立联系，来张图帮助大家理解。


[摘自：https://www.jianshu.com/p/6fc0601ffe34](https://www.jianshu.com/p/6fc0601ffe34)


```java

//空占位
static final Node EXCLUSIVE = null;

/**
 *  把当前线程封装成node置于队尾
 * @param mode固定是null
 **/
private Node addWaiter(Node mode) {
    //创建一个持有当前线程的node
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        //如果tail存在，就把新node置于队尾
        node.prev = pred;
        //变更tail为新创建的node
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

//通过设置头尾节点，建立新的队列顺序
private Node enq(final Node node) {
    for (;;) {
        //拿到缓存的tail
        Node t = tail;
        if (t == null) { 
            //如果没有缓存就初始化head和tail
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //如果已经缓存，
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

`addWaiter`方法的作用是**将当前线程封装成同步队列的节点，然后CAS快速入队**，并返回此节点。

1. 把线程封装成Node；
2. 新Node入队。

那么CAS入队失败，就需要调用`enq(Node node)`方法自旋入队。

这里就是一个性能策略，尽量的去避免自旋消耗cpu。

#### acquireQueued

```java
/**
*   用来阻塞node持有的线程，当其他线程释放锁的时* 候，才会唤醒
* @param node 新添加到队尾的node
* @param arg 写死为1
**/
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            //拿到当前node的上一节点
            final Node p = node.predecessor();
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                //获取锁成功，就把当前node置为head
                setHead(node);
                //打断链条，GC回收上一节点
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node))
                //阻塞线程，并检测线程是否被中断
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        //重置各种状态，并唤醒被阻塞的线程
        cancelAcquire(node);
        if (interrupted)
            //中断线程
            selfInterrupt();
        throw t;
    }
}

//阻塞线程，并检测线程是否被中断
private final boolean parkAndCheckInterrupt() {
    //阻塞线程，不会释放锁
    LockSupport.park(this);
    return Thread.interrupted();
}
```

最后AQS阻塞线程的秘密终于揭开了，在`acquireQueued(final Node node, int arg)`方法中，AQS通过调用`LockSupport.park(this)`来阻塞线程。通过搜索`LockSupport.unpark(this)`方法，发现在`release(int arg)`时会让下一个节点node唤醒。

### unlock()

同步队列（CLH）遵循FIFO，首节点是获取同步状态的节点，首节点的线程释放同步状态后，将会唤醒它的后继节点（next），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单。如下图：

![](/images/CLH3.jpg)

设置首节点是通过获取锁成功的线程来完成的（获取同步状态是通过CAS来完成），只能有一个线程能够获取到锁，因此设置头节点的操作并不需要CAS来保证，只需要将首节点设置为其原首节点的后继节点并断开原首节点的next（等待GC回收）即可。

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    //尝试释放锁
    if (tryRelease(arg)) {
        
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

//尝试释放锁
@ReservedStackAccess
protected final boolean tryRelease(int releases) {
    //检查状态机，是否可释放
    int c = getState() - releases;
    
    if (Thread.currentThread() != getExclusiveOwnerThread())
        //如果当前线程不是独占线程，则抛出异常
        throw new IllegalMonitorStateException();
    //改标记位
    boolean free = false;
    if (c == 0) {
        free = true;
        //释放独占线程
        setExclusiveOwnerThread(null);
    }
    //变更状态机
    setState(c);
    return free;
}

//唤醒下一个线程
private void unparkSuccessor(Node node) {

    int ws = node.waitStatus;
    if (ws < 0)
        //重置node的状态
        node.compareAndSetWaitStatus(ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }

    //如果存在next，唤醒下个node线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

其实我觉得这里就没必要细看了，细看也记不住。

`unlock()`方法主要是通过AQS的`release(int arg)`实现的，内部机制无非就是重置状态机，释放独占线程，并且唤醒下一个线程。而唤醒线程的方式也必然是`LockSupport.unpark(s.thread)`。

### 共享锁的申请

先回顾一下独占锁的申请：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //中断当前线程
        selfInterrupt();
}
```

线程申请独占锁失败会通过`acquireQueued`进入队列等待。

共享锁的申请则要简单的多：

```java
public final void acquireShared(int arg) {
    //调用抽象方法tryAcquireShared，可以看到返回了一个计数，如果计数大于0就表示成功拿到锁
    if (tryAcquireShared(arg) < 0)
        //如果返回小于0会调用doAcquireShared
        doAcquireShared(arg);
}

static final Node SHARED = new Node();

private void doAcquireShared(int arg) {
    //同样调用了addWaiter，但是传递的不是null而是一个空Node（），这样就保证Node.SHARED一直是[当前]节点的后继节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //拿到上一节点
            final Node p = node.predecessor();
        
            if (p == head) {
                //如果上个节点是head

                //再次尝试申请锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //如果拿到锁就释放head
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }

        
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                //阻塞线程，并检测线程是否被中断
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

先调用`addWaiter()`向队尾添加一个空的node并包装当前的thread，然后给这个空的节点增加后继节点`Node.SHARED`。再进入自旋，从当前节点向上遍历，如果当前节点的前驱节点是head，就尝试申请锁，申请成功就释放head，并把自己置于队首。其他位置的线程会判断状态是否应该被阻塞，然后再进入阻塞状态。

我们看到，**共享锁全程没有判断当前线程是否独占线程**，只要调用`tryAcquireShared()`方法不返回负数，就能成功申请到锁，而且同一时间可以多个线程并发拿到锁。

## 自定义一个最简单的重入锁

上面终于把源码巴拉完了，但是整个流程记不住。记不住怎么办？想办法，所以我决定自己实现一个最简单的可重入非公平锁：

```java
/**
 * 我自己实现的非公平锁
 */
public class MyLock {

    private final Sync sync;


    public MyLock() {
        this.sync = new Sync();
    }

    public void lock() {
        sync.acquire(1);
    }

    public void unlock() {
        sync.release(1);
    }

    /**
     * AQS实现类
     */
    class Sync extends AbstractQueuedSynchronizer {

        /**
         * 重写tryAcquire
         */
        @Override
        protected boolean tryAcquire(int acquires) {
            //拿到当前线程
            final Thread current = Thread.currentThread();
            //拿到AQS的状态机
            int c = getState();
            //判断状态机如果是初始状态
            if (c == 0) {
                //CAS由0到1
                if (compareAndSetState(0, acquires)) {
                    //当前线程设置为独占线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                //如果当前线程已经是独占线程，改变AQS状态
                int nextc = c + acquires;
                setState(nextc);
                return true;
            }
            return false;
        }

        /**
         * 重写tryRelease
         */
        @Override
        protected boolean tryRelease(int release) {
            int c = getState() - release;
            boolean free = false;
            if (c == 0) {
                free = true;
                //释放独占线程
                setExclusiveOwnerThread(null);
            }
            //改变状态机
            setState(c);
            return free;
        }

    }
}
```

从这里看出，利用AQS实现一个锁的流程：

1. 继承AQS；
2. 重写`tryAcquire`，当前线程置为独占线程，`setState()`变更状态；
3. 重写`tryRelease(int release)`，释放独占线程，`setState()`变更状态。

然后激动的试了一下：

```java
private static int num;
    private static MyLock myLock = new MyLock();

    public static void main(String[] args) throws Exception {

        Thread t1 = new Thread() {

            @Override
            public void run() {
                concat();
            }

        };

        Thread t2 = new Thread() {

            @Override
            public void run() {
                concat();
            }

        };

        t1.setName("t1");
        t2.setName("t2");

        t1.start();
        t2.start();
    }

    private static void concat() {
        try {
            myLock.lock();
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(3000);
                    num++;
                System.out.println("线程" + Thread.currentThread().getName() + "把num改为" + num);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        } finally {
            myLock.unlock();
        }
    }
```

最后惴惴不安的看了下结果：

```java
线程t1把num改为1
线程t1把num改为2
线程t1把num改为3
线程t1把num改为4
线程t1把num改为5
线程t1把num改为6
线程t1把num改为7
线程t1把num改为8
线程t1把num改为9
线程t1把num改为10
线程t2把num改为11
线程t2把num改为12
线程t2把num改为13
线程t2把num改为14
线程t2把num改为15
线程t2把num改为16
线程t2把num改为17
线程t2把num改为18
线程t2把num改为19
线程t2把num改为20
```

没问题，i did it !

## AQS的条件队列

### 条件队列的使用VSObject监视器方法

#### 对象监视器锁使用

```java
public class App {

    // 创建一把锁
    private static Object obj = new Object();

    public static void main(String[] args) {

        Thread t1 = new Thread() {
            @Override
            public void run() {
                //上锁
                synchronized (obj) {
                    try {
                        System.out.println(getName() + "开始等待");
                        //阻塞等待
                        obj.wait();
                        System.out.println(getName() + "已被唤醒");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t1.setName("t1");

        Thread t2 = new Thread() {
            @Override
            public void run() {
                //上锁
                synchronized (obj) {
                    try {
                        System.out.println(getName() + "已开始执行，3秒后将唤醒t1");
                        sleep(3000);

                        //唤醒
                        obj.notify();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t2.setName("t2");

        t1.start();
        t2.start();
    }
}
```

用法解析：

- 阻塞`wait`和唤醒`notify`都必须在持有锁的前提下调用，否则会报异常；
- 通过`wait()`阻塞线程；
- 通过`notify()`唤醒线程；
- 阻塞`wait`之后不会再执行任何代码，所以`notify`不应该在`await`之后。

输出：

```java
t1开始等待
t2已开始执行，3秒后将唤醒t1
t1已被唤醒
```

#### 条件锁使用

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class App2 {

    // 创建一个独占锁
    private static ReentrantLock lock = new ReentrantLock();

    // 用独占锁创建Condition
    private static Condition condition = lock.newCondition();

    public static void main(String[] args) {
        Thread t1 = new Thread() {
            public void run() {
                try {
                    lock.lock();
                    System.out.println(getName() + "开始等待");
                    condition.await();
                    System.out.println(getName() + "已被唤醒");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            };
        };
        t1.setName("t1");

        Thread t2 = new Thread() {
            public void run() {
                try {
                    lock.lock();
                    System.out.println(getName() + "已开始执行，3秒后将唤醒t1");
                    sleep(3000);
                    condition.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            };
        };
        t2.setName("t2");

        t1.start();
        t2.start();
    }
}
```

用法解析：

- 不论阻塞还是环境，都需要通过`lock.lock()`获取锁，否则或报监视器异常；
- 使用`await()`方法阻塞线程；
- 使用`signal()`方法唤醒线程；
- `await()`阻塞之后不会执行任何代码，直到被唤醒。

输出：

```java
t1开始等待
t2已开始执行，3秒后将唤醒t1
t1已被唤醒
```

从上面的对比可以看出，条件锁与对象监视器锁的用法几乎一样：

| Object        | Condition     | 作用         |
| ------------- | ------------- | ------------ |
| `wait()`      | `await()`     | 阻塞线程     |
| `notify()`    | `signal()`    | 唤醒线程     |
| `notifyAll()` | `signalAll()` | 唤醒全部线程 |

但是`Condition`在使用起来**更加灵活，且在同一把锁中支持多个条件**，所以应该优先使用。

### ReentrantLock VS synchronized

| 区别     | synchronized           | ReentrantLock        |
| -------- | ---------------------- | -------------------- |
| 来源     | jvm                    | JUC包                |
| 原理     | 对象监视器             | AQS队列+CAS+valatile |
| 易用     | 强                     | 弱                   |
| 释放     | 自动                   | 手动unlock           |
| 位置     | 对象、方法、类、代码块 | 只有代码块           |
| 可中断   | 不                     | 可                   |
| 公平锁   | 不支持                 | 支持                 |
| 精确唤醒 | 不支持                 | 支持                 |
| 多条件   | 不支持                 | 支持                 |
| 结果     | 只能一直阻塞           | 可返回结果           |
| 超时     | 不支持                 | 支持                 |

### 条件队列原理

#### `newCondition()`

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

很简单，就是构建了一个`ConditionObject`对象，他是`Condition`的实现类。

#### `await()`阻塞方法

```java
public final void await() throws InterruptedException {
    //当前线程的中断检查，会重置中断状态
    if (Thread.interrupted())
        throw new InterruptedException();
    
    Node node = addConditionWaiter();

    //释放之前获取到的锁资源，因为后续会阻塞该线程，所以如果不释放的话，其他线程将会等待该线程被唤醒
    int savedState = fullyRelease(node);
    int interruptMode = 0;

    //isOnSyncQueue方法会遍历CLH队列寻找当前node
    while (!isOnSyncQueue(node)) {
        //如果当前节点不在CLH队列中则阻塞住，等待unpark唤醒
        //阻塞当前线程
        LockSupport.park(this);
        //中断检测
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //唤醒后，会再次调用acquireQueued来阻塞，并且会不断的tryAcquire队首，让队首尝试获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

//把当前线程包装成一个新的node，并且设置成等待条件状态，加入条件队列
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //当前线程包装成一个新node，状态为：等待条件
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    
    if (t == null)
        firstWaiter = node;
    else
        //node的nextWaiter是一条单向链，这里是把新的节点加入条件队列中
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}

//清除条件队列中已不是等待条件状态的节点
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            //释放队列中所有状态不是条件的节点
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}

final boolean isOnSyncQueue(Node node) {
    //如果node是condition状态或者没有前驱节点返回false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //如果存在下一个节点，返回true，会终止自旋
    if (node.next != null) // If has successor, it must be on queue
        return true;

    return findNodeFromTail(node);
}

//判断当前节点是否存在于CLH队列中
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

#### `signal()`唤醒方法

```java
public final void signal() {

    //如果当前线程不是独占线程，就会抛异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    
    Node first = firstWaiter;
    if (first != null)
        //唤醒线程，改变状态
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        //剔除当前节点
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}

//把node的状态从CONDITION变为其他状态，唤醒线程
final boolean transferForSignal(Node node) {

    //CAS操作重置node的状态
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //走到这里说明该节点的状态已经被修改成了初始状态0。把其加入到CLH队列尾部，并返回前一个节点
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        //唤醒线程
        LockSupport.unpark(node.thread);
    return true;
}
```

`isHeldExclusively` 当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占。这里看出调用`signal()`时必须位于独占线程。

由以上代码可以看到：

- `signal()`唤起线程时，会把线程从条件队列中剔除；
- `await()`方法中阻塞的线程被唤起时会调用`acquireQueued()`方法不断获取锁。

## 全文总结

### 一句话总结AQS原理

AQS内部包含`int state`状态机和CLH同步队列，通过判断状态机决定线程是否能获得锁，并利用CAS原子操作改变状态，并让拿不到锁的线程加入CLH同步队列，依赖`LockSuport.part`方法阻塞线程，当锁释放时会唤醒线程，让队首节点出队，并让下一个线程自旋竞争锁。

同时AQS利用Node中的`nextWaiter`单向链作为条件队列，让其中符合条件的节点加入CLH队列获得竞争锁的机会。

### 可重入锁的原理

ReentrantLock是在AQS的基础上重写了`tryAcquire()`方法，判断当前线程是否是独占线程，是的话就直接让当前线程获得锁。