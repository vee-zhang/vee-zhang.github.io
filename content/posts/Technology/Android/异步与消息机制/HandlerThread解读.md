---
title: HandlerThread解读
date: 2021-03-18 14:22:41
tags:
---

## 用法

```java
// 步骤1：创建HandlerThread实例对象
// 传入参数 = 线程名字，作用 = 标记该线程
HandlerThread mHandlerThread = new HandlerThread("handlerThread");
// 步骤2：启动线程
mHandlerThread.start();

// 步骤3：创建工作线程Handler & 复写handleMessage（）
// 作用：关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
// 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
Handler workHandler = new Handler(mHandlerThread.getLooper()) {
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);

        Log.d("handler", "收到消息: " + msg.obj);

        // 注意，不能在此处修改UI,因为workHandler还是位于工作线程，需要切换到主线程去修改UI
    }
};

Message msg = Message.obtain(workHandler);

msg.obj = "我发了一条消息"; // 消息的存放
msg.sendToTarget();

// 步骤5：结束线程，即停止线程的消息循环
mHandlerThread.quit();
```

## 原理

```java
//继承Thread，本身就是线程
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;

    //内部持有looper
    Looper mLooper;

    //内部持有handler
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        //Process是进程的帮助类，可提供进程信息
        mTid = Process.myTid();

        //初始化loop
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            //完成之后唤醒需要使用looper的线程，主要是针对getLooper()方法
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();

        //开启循环
        Looper.loop();
        mTid = -1;
    }
    
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // 确保Looper一定能实例化
        synchronized (this) {

            // 自旋
            while (isAlive() && mLooper == null) {
                try {
                    // 阻塞调用的线程，直到run方法中looper初始化完成
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    //初始化内部Handler，但是外部不能调用此方法
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    //关闭
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    //安全关闭
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    //返回线程id
    public int getThreadId() {
        return mTid;
    }
}
```

## 总结

- `HandlerThread`本身就是一个线程，只不过它持有`Looper`,在`run()`方法内部实现了`Looper.prepary()`和`Looper.loop()`。所以任务也只能串行。
- 由于Looper是异步创建的，通过锁机制配合`wait/notifyAll`实现了一个阻塞唤醒机制，保证Looper在异步状态下必定能创建成功。
- 使用完要手动退出，否则线程一直存在。
