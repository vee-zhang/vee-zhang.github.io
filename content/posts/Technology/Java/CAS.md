---
title: CAS原理和问题
date: 2021-01-07 16:30:36
tags: [Java]
---

## 死锁

死锁是指多个的线程在执行过程中，由于竞争资源或者由于批次通信而造成的一种阻塞现象。若无外力作用，他们都将无法推进下去。

## 悲观锁与乐观锁

- 悲观锁: 假定会发生并发冲突，即共享资源会被某个线程更改。所以当某个线程获取共享资源时，会阻止别的线程获取共享资源。也称**独占锁或者互斥锁**，例如java中的synchronized同步锁。
- 乐观锁: 假设不会发生并发冲突，只有在最后更新共享资源的时候会判断一下在此期间有没有别的线程修改了这个共享资源。如果发生冲突就**重试**，直到没有冲突，更新成功。CAS就是一种乐观锁实现方式。

由于乐观锁不会阻塞其他线程，所以相对相率更高。

## CAS

### 定义 compare and swap 比较并且交换

JVM的CAS同时包含了CPU的CAS指令和自选操作。

CPU的CAS指令核心是三个值之间的比较和交换：

- 当前内存值(O)
- 预期原来的值(C)
- 期待更新的值(U)

比较方式：

如果内存位置O的值与预期原值C相匹配，那么处理器会自动将该位置值更新为新值,返回true。否则处理器不做任何操作，返回false。

JVM接收到false后会自旋操作，直到收到true为止。

eg:


假设当前有两个线程t1和t2并发操作O（O处于内存中），当t1拿到锁准备开始变更O时，t1会先把O拷贝出一个它自己的线程副本C(处于CPU缓存中)，然后对副本做对应的操作产生U，之后再用C去跟内存中的O做比较，如果C=O，说明内存中的O没有被其他线程变更，那么就把U刷写进内存替换掉O，并且返回true；如果C!=O，就直接返回false。

JVM接收到false后，就会从头再来一遍，也就是自旋。但是要注意的是，此时内存中的O已经被t2更改为O+了,所以t1会把O+作为当前值创建副本C+,执行操作后产生U+,然后用C+再去跟内存中的值对比。如此循环往复，直到CPU返回true为止。

典型就是JUC并发框架下的atomic原子类型。

注意：CAS本身不循环，循环也就是自旋是由JVM实现的。

### CS的三大问题

- ABA问题
- 开销问题
- 只能保证一个内存变量的原子操作

#### ABA问题

问题描述：线程t1将它的值从A变为B，再从B变为A。同时有线程t2要将值从A变为C。但CAS检查的时候会发现没有改变，但是实质上它已经发生了改变 。可能会造成数据的缺失。

##### 问题描述：

t1和t2同时操作A，t1要把A改为B再改回A，并且t1比t2速度快的多，导致t2要把A改为C时，比较的结果会认为A没有被改变过，会直接进行替换，导致了数据丢失。

##### eg

有一条单向链A->B，我们想通过并发把它变为B->C->D。

线程t1负责把A移除，让栈顶变为B，但是还没有进行CAS。

此时一个非常快的线程t2，它把链条中的元素全部出栈，然后再依次装入A、C、D

此时单向链变为A->C->D

然后t1在进行CAS时，由于栈顶还是A，所以compare就会通过，并且完成替换。

此时单向链变为：B->null

从结果上来看，C和D丢失掉了。

##### 解决办法

- 加时间戳或者版本戳
- AtomicStampedReference<E>（内部也是版本戳）

AtomicStampedReference怎么用暂时还不会，以后再说吧😄

```java
public static void main(String[] args) {
    AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100,1);

    //飞快的t2
    new Thread(() -> {
        stampedReference.compareAndSet(100, 101, stampedReference.getStamp(), stampedReference.getStamp() + 1 );
        stampedReference.compareAndSet(101, 100, stampedReference.getStamp(), stampedReference.getStamp() + 1 );
    },"t2").start();

    //龟速的t1
    new Thread(() -> {
        int stamp = stampedReference.getStamp();
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        boolean b = stampedReference.compareAndSet(100, 2019, stamp, stamp + 1);

    },"t1").start();

}
```

#### 开销问题

自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。貌似目前无解，好像也很难遇到。

#### 只能保证一个共享变量的原子操作

- 用锁
- 多个变量合并成一个变量

## AotomicInteger原理

```java
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}

//CAS操作
private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
public final boolean compareAndSet(int expect, int update) {
    return U.compareAndSwapInt(this, VALUE, expect, update);
}
```

可以看到，这里使用`do-while`对CAS做判断，如果**CAS返回false，那么就继续循环**。