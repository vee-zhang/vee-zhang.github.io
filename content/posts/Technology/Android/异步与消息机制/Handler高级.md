---
title: Handler的原理
date: 2021-03-04 10:40:11
tags:
---

## Handler的原理

内存共享方案，并且通过加锁避免线程之间的相互干扰。

>为何不用wait/notify

答案：

通过阅读`main()`函数，会发现主线程完全就是基于loop来运行的，所以的Android完全就是在handler+loop+message上面玩的。那么Handler机制最核心的用途是：**维持程序的运行！**，而消息机制反而是次要用途。

## MessageQueue的数据结构

由单链表实现的优先级队列。

优先级队列，根据时间优先级的插入排序算法的单链表。

## ThreadLocal

ThreadLocal是线程上下文的存储便变量，

[] 需要看一下ThreadLocal源码

## 面试题

1. 一个线程有几个Handler?
    无数个。
2. 一个线程有几个Looper?如何保证？
   1个，ThreadLocal.
3. Handler内存泄露的原因？
   匿名创建时，持有外部Activity的强引用，导致Activity不能GC。
4. 在子线程new Handler();
    需要自己调用Looper.prepary()和Looper.loop()。
5. 子线程中维护的Looper当无消息的时候如何处理？
   需要自己`Looper.quit()`，否则线程一直运行。
6. 生产者消费者模型中的阻塞机制
    使用epoll机制，当msg没到时间时，会epoll睡眠一段时间；当没有msg时，睡眠默认时间（很长）。当调用到enqueueMessage时，会调用nativeWake来唤醒。
7. 消息队列无消息如何处理？
    epoll机制睡眠，当`enqueueMessage()`时通过`nativeWake()`唤醒。整个过程是阻塞进行的。在子线程中，需要自己调用quit退出。
8. 多个Handler向MessageQueue中添加数据，但是他们处于不同的线程中，怎么确保数据安全？
   `sysnchronized`锁了this，锁的是当前的`MessageQueue`对象，他的所有的函数和代码块都会受限。
9. 主线程是否允许quit？
    主线程不允许，一旦调用就会抛异常。Zygote会fork自身，给每一个应用创建JVM，然后启动AcitivityThread，在后者执行的main方法中会调用`Looper.preparyMainLooper()`和`Looper.loop()`，应用的所有的代码都是在这个loop方法中运行的。
10. 夸线程通信的原理？
    通过共享内存实现。
11. 为什么线程间不会干扰？
    通过锁。
12. 为什么不用wait/notify?
    因为Java的wait/notify底层调用c++实现，Android自己实现了一个基于linux的epoll机制，效率更高？



## TODO

1. ThreadLocal原理进一步了解
