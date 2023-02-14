---
title: "IntentService"
subtitle: ""
date: 2022-03-28T22:47:09+08:00
lastmod: 2022-03-28T22:47:09+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: []

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---
面试题：场景需要在子线程中依次执行多个子任务，要保证任务时序，所有子任务执行完毕该线程自动退出，怎么实现？
<!--more-->
IntentService扩展自Service，是Service和HandlerThread的结合体，天生就在自线程中，且完成任务后自动停止，适合处理自线程耗时任务。如果多次启动IntentService，则每一个任务会以队列的方式在`onHandlerIntent`方法中依次执行。

## 原理

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    @UnsupportedAppUsage
    private volatile ServiceHandler mServiceHandler;
    private String mName;

    // 子线程Handler
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            //收到消息后调用onHandleIntent方法
            onHandleIntent((Intent)msg.obj);
            //任务执行完毕后关闭自己，注意这里的参数
            stopSelf(msg.arg1);
        }
    }

    /**
     * 构造方法需要传递线程名称
     */
    public IntentService(String name) {
        super();
        mName = name;
    }


    @Override
    public void onCreate() {

        // 初始化一个HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();

        // 用子线程的Looper初始化了子线程的Handler
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    

    /**
     * 直接调用onStart方法
     */
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    /**
     * 主要是把任务封装成消息，发送给子线程处理
     */
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * 退出时关闭子线程Looper，使线程终止
     */
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    /**
     * 一般不需要用这个方法，也可以通过重写实现
     */
    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * 我们需要重写这个方法去实现业务逻辑
     */
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

原理很简单：

在创建service时，`onCreate`方法是必须只调用一次的，所以在这里初始化了一个`HandlerThread`和归属于他的Handler。

当我们在activity中每次调用`startService()`，会回调service的`onStartCommand()`方法，在其中调用了`onStart()`，把intent封装成消息发送给子线程。由于消息没有`when`参数限制延迟时间，所以消息在`MessageQueue`中是按照FIFO(先进后出)的顺序排列的。

当消息经过Looper传递到Handler的`HandlerMessage()`方法时，会调用`onHandleIntent()`方法由我们自己去处理。处理完之后会尝试销毁service。

在销毁service时传递了startID，此时系统会检查service中是否还有其他的startID，如果有的话就不会退出，好让剩余的任务继续执行；如果没有，则会销毁Service.