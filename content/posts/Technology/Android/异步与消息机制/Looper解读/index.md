---
title: Looper解读
date: 2021-03-02 17:30:46
tags: [Android]
---

## 类图

![类图](Looper.dotuml.svg)

## prepare

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    // 如果线程已经有了Looper就会报错
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

/**
 * 构造方法中创建MessageQueue，并引用当前线程。
 */
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

`Looper.prepare()`方法只是为了创建`MessageQueue`和初始化`ThreadLocal`。

之前了解过，`ThreadLocal`可以保证多线程访问共享变量的线程安全问题。他不像`synchronized`靠阻塞实现线程安全，而是通过对变量拷贝的方式，使每一个线程都操作自己的拷贝，实现线程安全，所以效率要优于`synchronized`。[详情看这里](https://www.jianshu.com/p/6fc3bba12f38)和[这里](http://www.jasongj.com/java/threadlocal/)

ThreadLocal在此处的使用是为了保证每个线程都只能分配到一个Looper对象，多次调用`Looper.prepary()`方法会抛出异常。而且在其他线程去操作这个Looper对象可以保证线程安全，比如在主线程中调用`mHandler.getLooper().quitSafely()`来终止子线程的Looper对象的`loop()`循环。

## loop

只看核心：

```java
public static void loop() {
    final Looper me = myLooper();//其实就是从threadLocal中取出looper
    //省略判空

    me.mInLoop = true; //改变状态

    final MessageQueue queue = me.mQueue;//拿到MessageQueue

    ...

    //接下来就是启动个死循环，为什么不用while而用for呢？
    for (;;) {
        Message msg = queue.next(); // 从MessageQueue中取出Message
        if (msg == null) {
            //如果池子里面没有msg了则终止循环，所以这里并不是一个真正的死循环
            return;
        }
        try {
            msg.target.dispatchMessage(msg);//调用Handler的dispatchMessage
        } catch (Exception exception) {
            //...
        } finally {
            //...
        }

        //回收消息
        msg.recycleUnchecked();
    }
```

这里死循环的写法比较特殊，为什么不用`while(true)`或者`do-while(true)`呢？一开始总是想不明白。后来发现，用while系列循环，无论如何都要传递个表达式进去，那么就涉及到了内存占用和CPU运算。而用空for循环，则不必创建任何变量或结构体，能够最大限度的降低内存和CPU占用，真是学到了。

今天又突然想到，既然涉及到表达式，那么每次循环的时候都需要判断表达式是否成立，无可避免浪费了cpu，同时带来了cpu计算单元做无效计算带来的性能损失。

上面的代码中可以看到，当没有msg时，loop就会退出，那么问题来了，什么时候会没有msg呢？并且一旦loop退出，那么主线程不也就跟着退出了吗？目前我的猜想是，当没有消息时，Android通过epool机制让cpu休眠，当有了新的msg时又会唤醒cpu，从而保证一直存在msg。而休眠的时机是在`messageQueue`的`next()`方法中进行的：

```java
if (msg != null) {
    ...
} else {
    //如果当前没有消息，把唤醒时钟设为-1，那么意味着线程会沉睡Integer.MaxValue
    nextPollTimeoutMillis = -1;
}
```

## 主线程中为何不ANR也不会内存泄露

我们都知道线程一旦运行完毕就会回收，那么主线程中没有执行任何动作时为何不会回收呢？原因就是因为looper在死循环，阻塞了主线程的回收，那么相应的一旦不再死循环，程序也就退出了。Android同时利用Looper的死循环，发送消息，比如通知View重绘等等。

## 会不会阻塞子线程呢？

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        Looper.prepare();
        Handler handler = new Handler();
        Message msg = Message.obtain(handler, new Runnable() {
            @Override
            public void run() {
                Log.d("", "run: ");
            }
        });
        msg.sendToTarget();
        Looper.loop();

        Log.d("这里永远都不会执行", "run:");
        int i = 0;
    }
}
```

我用**Profile**持续观察之后发现：答案是会！CPU会一直被占用，线程池也无法回收该线程！

## 终止loop

所以为了尽可能减少CPU浪费，应该调用`handler.getLooper().quitSafely();`

```java
public void quitSafely() {
    mQueue.quit(true);
}
```

再来看`mQueue.quit(true)`：

```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

主要是修改了`mQuitting = true;`，这就是个标记，用来判断是否终止死循环。造成的效果呢当然是在`loop`方法的死循环中调用`mqueue.next()`时，会受到下面代码的影响：

```java
if (mQuitting) {
    dispose();
    return null;
}

private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}

private native static void nativeDestroy(long ptr);
```

直接返回null，而在Looper的`loop`方法的死循环中有这样的判断：

```java
...
Message msg = queue.next(); // might block
if (msg == null) {
    // No message indicates that the message queue is quitting.
    me.mLogging.println("注意啦，真的停啦～～～");
    return;
}
...
```

如果`queue.next`返回null那么就终止循环。

### 总结

`Looper`其实比较简单，核心就两个方法：

1. `prepary()`方法用来创建`MessageQueue`和`ThreadLocal`；
2. `loop()`方法就是永动机，不断从message中拿出msg来消费。
3. 只有调用`quitSafely()`或者`quit()`这两个方法修改`mQuitting`这个标记位，使MessageQueue的`next()`方法返回null，才能退出`loop()`方法。
4. 一个线程只有一个Looper，是通过ThreadLocal实现的。
