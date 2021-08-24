# HandlerThread解读


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
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
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

`HandlerThread`本身就是一个线程，只不过它持有`Looper`,在`run()`方法内部实现了`Looper.prepary()`和`Looper.loop()`。所以任务也只能串行。

使用完要手动退出，否则线程一直存在。

