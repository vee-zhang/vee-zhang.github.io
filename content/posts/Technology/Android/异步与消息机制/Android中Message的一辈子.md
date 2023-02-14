---
title: Android中Message的一辈子
date: 2021-03-09 20:51:17
tags: [Android, handler, Message, Looper]
---

## 初始化

```java
Looper.prepare();
Handler mHandler = new Handler(Looper.myLooper());
Looper.loop();
```

Looper.prepare()源码：

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Looper的构造方法：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

初始化阶段主要做了四件事：

1. 依赖ThreadLocal给当前线程创建Looper的独立对象；
2. 在Looper构造方法中创建MessageQueue；
3. 创建Handler对象;
4. 启动循环获取消息。

## 创建消息

```java

//方式1:
Message.obtain();

//方式2：
mHandler.obtainMessage();
```

## 发送

```java
//方式1:
mHandler.sendMessage(msg);

//方式2:
msg.sendToTarget();
```

最终调用`MessageQueue.enqueueMessage(Message msg, long when)`。

所谓的发送消息，其实是干了3件事：

1. 把消息的状态改为inUse；
2. 重组MessageQueue中的mMessages链条，把新消息放在队首；
3. 唤醒Looper所在线程，开始循环处理消息。

## 接收

Looper的`looper`方法中循环调用MessageQueue的`next`方法读取新的message，如果有新消息，就调用handler的`dispatchMessage(msg)`，进而调用`handleCallback()`或者调用`handleMessage`方法。最后通过msg的`msg.recycleUnchecked();`完成消息的回收复用。

而此时，如果没有消息，或者下一个消息是延迟消息且还没到时间，looper会回调所有的idleHandler，然后通过linux的epoll机制进入休眠，降低cpu负载，直到调用Looper对象的`quitSafely()`，进一步调MessageQueue的`quit()`方法，更改`mQuitting`标记，使MessageQueue的`next`方法返回null，跳出Lopper的`loop`循环，释放了线程。

> 明天补一张生命周期时序图吧，今天先到这了
