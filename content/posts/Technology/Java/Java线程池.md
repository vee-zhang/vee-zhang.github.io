---
title: Java线程池
date: 2021-03-18 15:33:07
tags:
---

## 阻塞队列 BlockingQueue

生产者&消费者模型

队列的意义：

- 生产者与消费者解耦
- 平衡生产与消费速度

>可应用于消息中心

默认实现：

|队列|界限|特点|
|---|---|---|
|ArrayBlockingQueue|有|一个由数组结构组成的有界|
|LinkedBlockingQueue|有|链表结构|
|PriorityBlockingQueue|无|支持优先级排序，可以传入比较器|
|DelayQueue|无|支持优先级，支持元素的延迟获取|
|SynchronousQueue||不储存元素，用来解耦|
|LinkedTransferQueue|无|链表结构|
|LinkedBlockingQueue||双向链表|

有界&无界

- 有界：长度有限，满了会阻塞；
- 无界：可以随意放东西，不会阻塞；

### 核心方法

　1.放入数据

- offer(anObject):表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则返回false.（本方法不阻塞当前执行方法的线程）；　　　　　　 
- offer(E o, long timeout, TimeUnit unit)：可以设定等待的时间，如果在指定的时间内，还不能往队列中加入BlockingQueue，则返回失败。
- put(anObject):把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.
- poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回null;
- poll(long timeout, TimeUnit unit)：从BlockingQueue取出一个队首的对象，如果在指定时间内，队列一旦有数据可取，则立即返回队列中的数据。否则知道时间超时还没有数据可取，返回失败。
- take():取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入; 
- drainTo():一次性从BlockingQueue获取所有可用的数据对象（还可以指定获取数据的个数），通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁。

#### 下面几个方法要记住

|方法|阻塞|作用|
|---|---|---|
|offer|no|添加元素成功返回true，如果队列已满返回false，重载方法可以设置等待时间。|
|put|yes|添加元素，如果空间已满，会阻塞线程直到添加成功|
|poll|no|取走队列首位元素，可以设定等待时间，失败返回false|
|take|yes|取走队列首位元素，如果没有就会阻塞线程直到有了为止|
|drainTo||一次性获取多个或全部元素|

## Java默认线程池

- `Executors.newCachedThreadPool()`，创建一个可缓存的无界线程池，如果线程池长度超过处理需要，可灵活回收空线程，若无可回收，则新建线程。当线程空闲超过60秒自动回收，当任务超过线程池的线程数，可无限创建新线程；
- `Executors.newFixedThreadPool(int nThreads)`，创建一个指定大小的线程池，可控制线程的最大并发数，超出的线程会在`LinkedBlockingQueue`阻塞队列中等待；
- `Executors.newScheduledThreadPool`，创建一个定长的线程池，可以指定线程池核定线程数，**支持定时及周期性任务**的执行；
- `Executors.newSingleThreadExecutor`，创建一个单线程的线程池。
- `Executors.newWorkStealingPool（java8引入）`，创建一个更加高效的线程池，不同于以上四种线程池，这个线程池拓展于`ForkJoinPool`，适合用于非常耗时或自任务众多的场景。


前四种是我们一般用到的线程池，都是拓展自ThreadPoolExecutor类。

## 自定义线程池与ThreadPoolExecutor

我们也可以通过使用`ThreadPoolExecutor`类实现自定义线程池。

```java
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    ThreadFactory threadFactory
)
```

参数解析：

1. corePoolSize
    线程池的核心线程数，默认情况下，核心线程会在线程池中一直存活，即使它们处于闲置状态。如果通过`allowCoreThreadTimeOut(boolean value)`方法设置为true，那么闲置的核心线程在等待新任务到来时会有超时策略，这个时间间隔由keepAliveTime所指定，当等待时间超过keepAliveTime后，核心线程就会被终止。
2. maximumPoolSize 
    线程池所能容纳的最大线程数，当活动线程数达到这个数值后，后续的新任务将会被阻塞。
3. keepAliveTime
    非核心线程闲置时的超时时长，超过这个时长，非核心线程就会被回收。当ThreadPool-Executor的allowCoreThreadTimeOut属性设置为true时，keepAliveTime同样会作用于核心线程。
4. unit
    用于指定keepAliveTime参数的时间单位，这是一个枚举，常用的有TimeUnit. MILLISECONDS（毫秒）、TimeUnit.SECONDS（秒）以及TimeUnit.MINUTES（分钟）等。
5. workQueue
    线程池中的阻塞队列，用来容纳超出core线程数的runnable。
6. threadFactory
    线程工厂，为线程池提供创建新线程的功能。ThreadFactory是一个接口，它只有一个方法：Thread newThread(Runnable r)，一般使用默认的，也可以自定义用来设置thread-name。
7. Rejected-ExecutionHandler handler 
    当线程池无法执行新任务时，比如线程全忙并且队列已满，这个时候ThreadPoolExecutor会调用handler的rejectedExecution方法来通知调用者，默认情况下rejectedExecution方法会直接抛出一个RejectedExecution-Exception。ThreadPoolExecutor为RejectedExecutionHandler提供了几个可选值：CallerRunsPolicy、AbortPolicy、DiscardPolicy和DiscardOldestPolicy，其中AbortPolicy是默认值。
    

ThreadPoolExecutor执行任务时大致遵循如下规则：

1. 如果当前线程数小于core，那就创建线程直接执行；
2. 如果core满了，就会把task放入BlockingQueue中排队；
3. 如果BlockingQueue也满了，就新建线程直接执行；
4. 如果当前线程数量超过maxPoolSizie，就拒绝。

默认实现的拒绝策略：

- AbortPolicy 直接抛出异常（抛异常）；
- DiscardPolicy 把新的任务直接扔掉（后进先出）。
- DiscardOldestPolicy 直接丢弃最旧的还没执行的任务（先进先出）；
- CallerRunsPolicy 让调用者所在线程去执行任务，比如调用者是主线程，就会切回主线程（扯皮）；

### 参数配置

AsyncTask支持并发时的配置：

通过以下代码获取CPU核心数：

```java
Runtime.getRuntime().availableProcessors()
```

#### 配置原则：

- CPU密集型：大规模运算，没有阻塞，为减少cpu轮转次数，尽量减少线程数量，公式：`CPU核心数+1`;
- IO密集型：阻塞多，尽量多给线程提高速度，公式：`CPU核心数*2`；
- 混合型：如果密集计算耗时与IO耗时差不多，就拆分成两个线程池来玩。

AsyncTask的配置：

- 核心线程数等于CPU核心数+1；
- 线程池的最大线程数为CPU核心数的2倍+1；
- 核心线程无超时机制，非核心线程在闲置时的超时时间为1秒；
- 任务队列的容量为128。


>为什么要加1？因为在硬盘上有虚拟内存，为了在发生“页缺失”现象时，不让CPU核心闲置，所以要加1。

### 线程池的关闭

- shutDown 关闭线程池，然后尝试中断当前**没有执行任务的线程**；
- shutDownNow 尝试中断所有的线程。

## 原理

### execute任务调度：

```java
//当前线程数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

public void execute(Runnable command) {
    
    //当前运行的任务数
    int c = ctl.get();

    //如果小于核心数，就addWork
    if (workerCountOf(c) < corePoolSize) {
        //如果当前活跃线程数小于设置的核心线程数，直接执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    //如果大于核心数，并且线程池还在运行，就添加到队列中
    if (isRunning(c) && workQueue.offer(command)) {
        
        int recheck = ctl.get();
        //如果线程池已经停止运行，就移除任务
        if (! isRunning(recheck) && remove(command))
            //移除成功就执行拒绝策略
            reject(command);
        else if (workerCountOf(recheck) == 0)
            //线程池不在运行状态，或者移除失败，并且当前没有任务，就开启一个空任务
            addWorker(null, false);
    }
    //如果添加任务失败，就执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

`execute`方法主要用来调度任务，据当前原子计数器`c`的状态和任务数，

- 如果当前任务数小于核心线程数，就直接`addWorker`；
- 如果大于核心线程数，就把任务添加到阻塞队列；
- 如果队列已满，尝试`addworker`;
- 如果队列满，addWorker失败，则执行拒绝策略。

### addWorker线程调度

```java
//我才是真正的线程池子
private final HashSet<Worker> workers = new HashSet<>();

private boolean addWorker(Runnable firstTask, boolean core) {
    
    //使用cas失败自旋来保证线程竞争问题
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty())
            )
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //CAS操作
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //Worker是对runnable和Thread的装箱，构造方法会调用ThreadFactory生成一个Thread
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();//上锁，
            try {
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();

                    //添加进线程栈
                    workers.add(w);


                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                //core线程直接运行
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

`addWorker`主要用来调度线程。首先创建work，然后把work添加进线程池`workers`，并通过主锁`mainLock`保证线程池`workers`的安全。最后启动`worker`的线程。
### work

```java
//ThreadPoolExecutor的匿名内部类，满足AQS实现不可重入的锁，满足Runnable，内部包含Thread
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
{
    //可以序列化
    private static final long serialVersionUID = 6138294804551838833L;
    //线程
    final Thread thread;
    //任务
    Runnable firstTask;
    
    volatile long completedTasks;

    //构造方法，调用时从ThreadFactory创建新的线程
    Worker(Runnable firstTask) {
        setState(-1); 
        this.firstTask = firstTask;
        //注意这里，给线程传入的是当前Worker
        this.thread = getThreadFactory().newThread(this);
    }

    //线程运行时会执行这个
    public void run() {
        runWorker(this);
    }

    //尝试Acquire
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    //尝试释放
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    //尝试中断
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

`work`是ThreadPoolExecutor的内部类，是一个满足AQS的`Runnable`，在构造方法中通过factory创建一个线程并传入他自身，那么当`addWork`时会启动这个线程，而线程会执行他自己的`run`方法，run方法又调用了外层类ThreadPoolExecutor的`runWorker`方法。

`work`还提供了AQS操作和中断线程的方法，由此可见，`work`是Thread的**装饰者**。

### runWorker：复用线程

```java
final void runWorker(Worker w) {
    //拿到线程
    Thread wt = Thread.currentThread();
    //拿到work构造方法传入的runnable
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //循环执行任务
        //如果firstTask有值,就直接执行这个任务
        //如果没有具体的任务,就执行getTask()方法从队列中获取任务，当然这个操作是阻塞的
        while (task != null || (task = getTask()) != null) {
            w.lock();

            //检查线程池是否shutDown了，确保线程中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();

            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

`runWorker`方法主要是对线程的控制，会循环调用`getTask`方法从任务队列中取出任务交给线程执行，达到线程复用的目的。当线程池shutDown时会中断线程。

### getTask阻塞队列取值

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    //死循环
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        //当前任务数
        int wc = workerCountOf(c);

        // 如果当前任务数大于core，或者core线程也允许超时回收
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //如果当前任务数大于core就等着拿，超时就返回null；否则就阻塞等，一直等到有了为止
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

`getTask`主要用于从任务队列中取值，如果当前任务数大于核心线程数就直接拿，否则会阻塞等。

### 总结线程池的原理

线程池我们从`execute`方法入手，`execute`方法主要用来调度任务，据当前原子计数器`c`的状态和任务数，

- 如果当前任务数小于核心线程数，就直接`addWorker`；
- 如果大于核心线程数，就把任务添加到阻塞队列；
- 如果队列已满，还是`addworker`;
- 如果队列满，addWorker失败，则执行拒绝策略。

`addWorker`主要用来调度线程。首先创建`worker`，然后把work添加进线程池`workers`，并通过主锁`mainLock`保证线程池`workers`的安全。最后启动`worker`的线程。

**`work`是runnable的装饰者**,除了具备runnable功能特性之外还提供AQS操作和中断线程的功能。它内部持有一个线程，由于他自己是runnable，所以线程会执行他自己，并再它自己的`run`方法内调用线程池的`runWorker`方法。

`runWorker`通过循环调用`getTask()`方法取出阻塞队列中的runnable来执行，当线程池shutdown的时候会尽量中断线程。

而`getTask()`方法会从阻塞队列中取出首位任务。如果队列中有任务会poll等待取出，等待的时间为线程池参数`keepAliveTime`，否则就会调用take一直阻塞等待任务。

## retry标记

上面的`addWork`方法中出现了`retry:`标记，这个标记主要是用于在多重循环中标记需要中断的循环层，也可以用其他字母+：的方式：

```java
//声明中断标记
aaa:
//第一层循环f1
for (int i = 0; i < 100; i++) {
    //第二层循环f2
    for (int j = 0; j < 200; j++) {
        if（j == 100)
        //将使f1跳过并继续
        continue aaa;
    }
}
```
