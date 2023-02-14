---
title: AsyncTask解读
date: 2021-03-17 16:59:41
tags: [Android, AsyncTask]
---

## 使用

```java
public class MainActivity extends AppCompatActivity {
    TextView tv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        tv = findViewById(R.id.tv);
    }

    public void start(View v) {
        MyTask task = new MyTask(this.tv);
        task.execute("我是");
    }


    @SuppressWarnings("deprecation")
    private static class MyTask extends AsyncTask<String, Float, String> {

        WeakReference<TextView> tv;

        MyTask(TextView tv) {
            this.tv = new WeakReference<TextView>(tv);
        }

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            // 任务执行前回调
            tv.get().setText("即将开始任务");
        }

        @Override
        protected String doInBackground(String... strings) {
            //后台任务逻辑
            if (strings.length > 0) {
                try {
                    for (int i = 0; i < 5; i++) {
                        Thread.sleep(2000L);
                        //报告进度
                        publishProgress(0.2f * (i+1));
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return strings[0] + "结果";
            }
            return null;
        }

        @Override
        protected void onProgressUpdate(Float... values) {
            super.onProgressUpdate(values);
            tv.get().setText("任务已执行"+values[0]);
        }

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            //刷新UI
            tv.get().setText(s);
        }
    }
}
```

几个重要方法：

| 方法             | 线程   | 执行                        |
| ---------------- | ------ | --------------------------- |
| onPreExecute     | 主线程 | 任务开始前执行              |
| doInBackground   | 子线程 | 后台任务执行时调用          |
| onProgressUpdate | 主线程 | `publishProgress`调用时回调 |
| onPostExecute    | 主线程 | 任务完成后调用              |
| onCancelled      | 主线程 | `cancell`调用时回调         |

核心就这几个方法，只有`doInBackground`在子线程中调用。然后创建一个AsyncTask的实例，执行`execute`方法，并传入相应参数就开始执行了。

同时为避免内存泄露，需要用**弱引用**关联UI。

## 原理

### 构造方法

```java
public AsyncTask() {
    this((Looper) null);
}

public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}

/**
*   最终构造方法
**/
public AsyncTask(@Nullable Looper callbackLooper) {
    //拿到Handler，默认拿到主线程Handler
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);

    //定义Callable，其中调用了doInBackground方法
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //调用doInBackground
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                //发送Message
                postResult(result);
            }
            return result;
        }
    };

    //用FutureTask包装Callable，提供启动、取消、监听功能
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

从这里可以看出，Async的主要流程在构造方法中就已经定义好了。

其中有个mWorker是`WorkerRunnable`类型：

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

查看源码才发现，这玩意并不是一个Runnable，而是个Callable，google跟我开了个玩笑。

接下来就是用`FutureTask`包装Callable，这样就可以具备启动、取消、完成监听功能。接下来我们去看`execute`方法。

### execute

```java
//静态final类型的线程池
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

//默认的线程池
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

//方法重载
@MainThread
public static void execute(Runnable runnable) {
    sDefaultExecutor.execute(runnable);
}
```

这里可以看出`execute`方法只能在主线程中调用。这就是`AsyncTask`的局限性了：只能用于主线程。

`executeOnExecutor`方法传入了默认的线程池。这个线程池是在AsyncTask类首次加载时初始化的。它是个static final类型。

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    //改状态
    mStatus = Status.RUNNING;

    //在MainThread回调onPreExecute
    onPreExecute();

    mWorker.mParams = params;

    //真正的执行任务
    exec.execute(mFuture);

    return this;
}
```

从switch判断可以知道，AsyncTask是一次性的，不能复用，只有`PENDING`状态的AsyncTask才能运行。而状态是在类加载时定义的，而且不支持修改：

```java
@UnsupportedAppUsage
private volatile Status mStatus = Status.PENDING;
```

在这个方法里，首先改变了全局的状态为「执行中」，然后通过`exec.execute(mFuture)`真正开始执行线程。在这里查看源码，发现`FutureTask`这个类同时遵循`Runnable`和`Future`接口，所以他可以被线程池execute，也能接收到callable的返回值。这也是Callable转Runnable的一种方式。

然后来看AsyncTask的三种状态：

```java
public enum Status {
    //未执行
    PENDING,
    //执行中
    RUNNING,
    //执行完
    FINISHED,
}
```

### publishProgress

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

报告消息是基于Handler+message的形式实现的。

这个消息的`what`是`MESSAGE_POST_PROGRESS`这个常量。`obj`是new了一个`AsyncTaskResult<Progress>(this, values)`：

```java
private static class AsyncTaskResult<Data> {
    final AsyncTask mTask;
    final Data[] mData;

    AsyncTaskResult(AsyncTask task, Data... data) {
        mTask = task;
        mData = data;
    }
}
```

接收消息的Handler，依靠这个Handler切换回主线程:

```java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

### onPostExecute

这个方法是在构造方法的`WorkerRunnable`中的逻辑执行完毕，最后finally里面调用的。

```java
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        mTaskInvoked.set(true);
        Result result = null;
        try {
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            result = doInBackground(mParams);
            Binder.flushPendingCommands();
        } catch (Throwable tr) {
            mCancelled.set(true);
            throw tr;
        } finally {
            postResult(result);
        }
        return result;
    }
};
```

在这个方法当然也是通过发送消息的方式达到线程间通信的目的：

```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

最后在Handler的handleMessage中通过what做判断，如果是postResult，就调用`finish()`方法：

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

AsyncTask的流程已经通了，最后再来看一下它是怎么玩线程池的。

### AsyncTask工作线程池的实现

它这里是自定义了一个线程池代理：

```java
private static class SerialExecutor implements Executor {

    //可作为栈或队列使用，性能较高
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        //在队尾添加元素
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        //poll()拿出队首元素
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

这货实际上是`THREAD_POOL_EXECUTOR`这个线程池的代理，作用就是维护任务队列。

### 线程池参数

```java
//核心线程数，一直处于活动状态
private static final int CORE_POOL_SIZE = 1;
//最大线程数，当一个任务提交到线程池中时，如果线程数量达到了核心线程数，并且任务队列已满，不能再向任务队列中添加任务时，这时会检查任务是否达到了最大线程数，如果未达到，则创建新线程，执行任务，否则，执行拒绝策略。
private static final int MAXIMUM_POOL_SIZE = 20;

//活跃时间，线程池中大于核心线程数的那部分线程，在执行完任务之后，在线程池中存活的时间
private static final int KEEP_ALIVE_SECONDS = 3;

//线程工厂
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

public static final Executor THREAD_POOL_EXECUTOR;
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(), sThreadFactory);
    threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

首先执行这个静态代码块，初始化线程池。

我们看这里的线程池最大核心数只有1，那就是**只能串行**，最大线程数20，最多能有19个线程排队等待。

### 线程池拒绝策略

```java
//核心线程数&最大线程数
private static final int BACKUP_POOL_SIZE = 5;
private static final RejectedExecutionHandler sRunOnSerialPolicy =
            new RejectedExecutionHandler() {
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        android.util.Log.w(LOG_TAG, "Exceeded ThreadPoolExecutor pool size");
        synchronized (this) {
            if (sBackupExecutor == null) {
                sBackupExecutorQueue = new LinkedBlockingQueue<Runnable>();
                sBackupExecutor = new ThreadPoolExecutor(
                        BACKUP_POOL_SIZE, BACKUP_POOL_SIZE, KEEP_ALIVE_SECONDS,
                        TimeUnit.SECONDS, sBackupExecutorQueue, sThreadFactory);
                sBackupExecutor.allowCoreThreadTimeOut(true);
            }
        }
        sBackupExecutor.execute(r);
    }
};
```

如果被拒绝，就新创建一个线程池去执行，这都可以？

## 总结

AsyncTask类首次加载的时候初始化两个静态的线程池，`SerialExecutor`是个代理，内部维护着`Runnable`队列，实际通过调用threadPoolExecutor来执行任务。由于线程池不能保证线程执行的先后，所以AsyncTask通过加入外部队列的方式保障了任务执行的顺序。

AsyncTask的构造方法内部初始化了`Callable`，AsyncTask的主要流程都定义在`Callable`的`call()`方法内部，然后通过`FutureTask`包装`Callable`转换为`Runnable`传给线程池执行，同时可以取消任务或者获取返回值，最后通过内部的`Handler`发送消息切换回主线程。

由于内部的Handler在创建时传入`Looper.getMainLooper()`，所以默认线程就是主线程，并且`execute()`方法带有`@MainThread`注解，导致AsyncTask只能在主线程中使用。

AsyncTask初始化时状态为`pending`，在任务执行过程中变换为`Running`或`finished`，不可手动更改状态，导致AsyncTask不能复用。

### 如何正确配置线程池的参数

前面我们讲到了手动创建线程池涉及到的几个参数，那么我们要如何设置这些参数才算是正确的应用呢？实际上，需要根据任务的特性来分析。

- 任务的性质：CPU密集型、IO密集型和混杂型
- 任务的优先级：高中低
- 任务执行的时间：长中短
- 任务的依赖性：是否依赖数据库或者其他系统资源
- 不同的性质的任务，我们采取的配置将有所不同。在《Java并发编程实践》中有相应的计算公式。

通常来说，如果任务属于CPU密集型，那么我们可以将线程池数量设置成CPU的个数，以减少线程切换带来的开销。如果任务属于IO密集型，我们可以将线程池数量设置得更多一些，比如CPU个数*2。

PS：我们可以通过Runtime.getRuntime().availableProcessors()来获取CPU的个数。

作者：juconcurrent
链接：https://www.jianshu.com/p/7ab4ae9443b9
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
