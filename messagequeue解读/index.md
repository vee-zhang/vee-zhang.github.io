# MessageQueue解读


## 回顾

handler发送消息：

```java
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

Looper循环读取消息并发送给Handler:

```java
for (;;) {
    Message msg = queue.next(); // 从MessageQueue中取出Message
    
    try {
        msg.target.dispatchMessage(msg);//调用Handler的dispatchMessage
    } catch (Exception exception) {
        //...
    } finally {
        //...
    }

    //无论如何都会回收消息
    msg.recycleUnchecked();
}
```

## enqueueMessage

```java
Message mMessages;

boolean enqueueMessage(Message msg, long when) {

    synchronized (this) {
        if (msg.isInUse()) {//发送的消息必须是闲置状态的
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        if (mQuitting) { //mQuitting是一个状态，只有MessageQueue的quit方法调用时才这个状态才会为true
            msg.recycle();//回收
            return false;
        }

        msg.markInUse(); //改状态为使用中
        msg.when = when;//注意这里赋值
        Message p = mMessages;
        boolean needWake;

        //重点是时间的比较，如果新的msg的时间早于当前的message，那么就把新的msg放在头部。
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();//如果是异步消息
            Message prev;
            for (;;) {//又一个死循环，用来向下遍历队列，
                prev = p;//把当前的msg设为前一个msg
                p = p.next;//当前的msg指向下一个msg
                if (p == null || when < p.when) {//如果我们设置时间的早于msg的时间，那么就停止遍历
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        
        if (needWake) {
            nativeWake(mPtr);//唤醒线程处理消息
        }
    }
    return true;
}
```

简而言之，`enqueueMessage`的作用是采用`synchronized`同步阻塞的方式操作Message链条的头部，依据Message的延迟时间，打断链条重新排序，同时修改msg的状态，利用共享内存（mMessages）达到线程间通信的目的。

那么所谓的`Handler.sendMessage(msg)`是真的发送消息吗？并不是！它只是修改了Looper中持有的MessageQueue中的mMessages的链条顺序，然后等待`Looper`不断循环获取，再传递回`mHandler.dispatchMessage(msg)`而已。但这一改加一取，就能达到线程间通信的效果，并让我一直误以为是真的通过序列化之后发送出去。

>不看不知道，一看好鸡贼！

## next

```java
@UnsupportedAppUsage
Message next() {
    
    //当loop执行了quit后，程序可能会重启looper，就会返回到这里。
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; //只有在第一次遍历时是-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            //用来在线程要进入阻塞之前跟内核线程发送消息，防止用户线程长时间的持有某个对象
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);//告诉linux的epoll：你丫可以睡nextPollTimeoutMillis这么长时间了！直到handler发送消息调用`mQueue.enqueueMessage()`方法时会被唤醒，或者当前消息到时间时也会被唤醒。

        synchronized (this) {
            // 尝试检索下一个msg，找到就返回
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                //靠栅栏停滞，找到下一个异步msg
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //下一个消息还没到时间。设置一个唤醒时段
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {//如果不是异步消息，prevMsg就是null
                        mMessages = msg.next;//往上提一位
                    }
                    msg.next = null;//打断链条单独取出
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();//改变状态为使用中
                    return msg;//返回
                }
            } else {
                //如果当前没有消息，把唤醒时钟设为-1，那么意味着线程会沉睡Integer.MaxValue
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            ///接下来就是喜闻乐见的idleHandler了

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();//如果没有添加过idleHandler，那么就是0
            }
            if (pendingIdleHandlerCount <= 0) {
                //如果没有idleHandler需要执行，那就继续loop
                mBlocked = true;
                continue;
            }

            //初始化一个固定长度的数组
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 遍历执行所有的idleHandler.
        //只有在第一次循环时才会执行到这个代码块
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // 强行释放数组中的idleHandler，不会影响ArrayList中的idleHandler和当前正在使用的idleHandler。

            boolean keep = false;
            try {
                keep = idler.queueIdle();//回调queueIdle方法
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        //重置计数
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```

### IdleHandler

```java
/**
* Callback interface for discovering when a thread is going to block
* waiting for more messages.
* 当线程将要锁定（等待更多消息）时就会回调queueIdle方法
*/
public static interface IdleHandler {
    /**
    * 当MessqgeQueue已经处理完所有的message，并且将要进入睡眠和等待时这个方法才会被调用。
    * 返回true可以使IdleHandler保持激活状态，那么当下次线程空闲了还会调用，返回false就会移除IdleHandler。
    */
    boolean queueIdle();
}
```

原来这玩意根本不是什么Handler的子类，而是MessageQueue的一个内部静态类而已！

然后我就不太明白这个玩意了，去百了个度，拿到答案如下：

之前做过冷启动优化，在冷启动的场景有很多的任务其实并不需要马上启动，通常的做法就是做一个延迟启动，如下所示

```java
Handler mHandler = new Handler();
mHandler.postDelayed(() -> {
    //do something
}, 1000);
```

将任务延迟启动1000ms，但是这个延迟启动的时间不好确定，只能是自己预估的，对于一些高端手机1000ms可能多了，一些低端手机可能1000ms还不够。这个时候IdelHandler就可以解决这个问题，它能够在CPU空闲的时候再执行指定的任务。

使用方法也很简单，如下所示，调用addIdleHandler方法就可以了:

```java
MessageQueue.IdleHandler idleHandler = new MessageQueue.IdleHandler() {
    @Override
    public boolean queueIdle() {
        //do something
        return false;
    }
};
Looper.getMainLooper().getQueue().addIdleHandler(idleHandler);
```

引自[](https://blog.csdn.net/hbdatouerzi/article/details/104148722)

>我惊叹啊，还能这么玩啊～～想到以前面试官问过我怎么做延迟启动，我回答ContentProvider和StartUp，我觉得是没错的，但是她想要的应该就是这个IdleHandler。

```java
public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}
```

添加时有非空判断和线程安全。

然后具体的操作也是在MessqgeQueue的`next`方法中进行的。那么IdleHandler怎么判断线程是否空闲呢？看`next`方法中这一部分：

```java
//条件1:没有msg或msg还没到执行的时候
if (msg != null) {
    if (now < msg.when) {
        //msg时间晚于当前时间
        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
    } else {
        // Got a message.
        mBlocked = false;
        if (prevMsg != null) {
            prevMsg.next = msg.next;
        } else {
            mMessages = msg.next;
        }
        msg.next = null;
        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
        msg.markInUse();
        return msg;
    }
} else {
    // No more messages.
    nextPollTimeoutMillis = -1;
}

//条件2:looper没有停止
if (mQuitting) {
    dispose();
    return null;
}

//条件3:不存在IdleHandler的话就重新遍历
if (pendingIdleHandlerCount <= 0) {
    // No idle handlers to run.  Loop and wait some more.
    mBlocked = true;
    continue;
}
```

所以判断线程是否将进入空闲，也就是IdleHandler的执行时机，是依据：

1. MessageQueue中没有消息或当前消息不必马上执行；
2. Looper没有释放；
3. 队列中存在IdleHandler。

