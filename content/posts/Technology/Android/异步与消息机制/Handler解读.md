---
title: Handler解读
date: 2021-03-04 10:40:11
tags:
---

## 构造方法

```java
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

构造方法中可以看到，`handler`持有`Looper`的`MessageQueue`。

## obtainMessage

```java
public final Message obtainMessage(){
    return Message.obtain(this);
}
```

Handler的`obtainMessage()`其实还是调用`Message.obtain()`方法，所以直接调用后者反而效率更高。那么来看一下`Message.obtain()`干了什么：

```java
public static Message obtain(Handler h) {
    Message m = obtain();
    m.target = h;

    return m;
}
```

没错，就是设置了一个**target**。

## sendMessage

```java
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

/**
* 核心在这里
*/
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
    msg.target = this;
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);//最后调用的是queue.enqueueMessage
}
```

`sendMessage`最后是通过`MessageQueue`的`enqueueMessage`实现消息的发送。那么我知道`message`有个`sendToTarget`方法，其实也是通过handler去发消息的。

```java
public void sendToTarget() {
    target.sendMessage(this);
}
```

**当消息发出后，就进入了生产者模型中。**

## post

post其实也是基于消息实现的，但与发送消息不同的是，`post`是先创建个消息，然后把`Runnable`类型的参数赋值给msg的`callback`。前面解读`Message`的时候了解到，`callback`是在handler接收到消息的时候才执行，它与handler在同一线程。

```java
/**
*   把Runnable添加到message中，并且在handler所在线程运行
*/
public final boolean post(@NonNull Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();//从message回收池中取一个闲置msg
    m.callback = r;//赋值callback
    return m;
}
```

## postDelayed

```java
public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}

public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

一般面试的会考这个方法，我感觉也没啥啊，无非就是个`SystemClock.uptimeMillis()`加上设定的延迟时间。

`SystemClock.uptimeMillis()`是从系统启动后开始累计的时间，并且在系统进入深度睡眠的时候暂停累计。

## 接收消息

前面在[Looper解读](Looper解读.md)里了解到，Looper在`loop`方法中死循环读取message，然后通过`msg.target.dispatchMessage(msg);`方法发送消息：

```java
/**
* Handle system messages here.
*/
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

这里就可以看到，**如果msg的callback不为空，那么就处理callback,反之则处理msg**。这就是为什么给msg添加了callback之后，Handler的`handleMessage`不会再回调的原因。

handleCallback:

```java
private static void handleCallback(Message message) {
    message.callback.run();
}
```

直接在当前线程去跑Runnable类型的callback。
