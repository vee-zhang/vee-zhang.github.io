---
title: Java线程
date: 2021-03-19 09:44:50
tags:
---

## 状态

- new 新建
- Runnable 可运行
- Blocaked 阻塞
- Waiting 等待
- Timed waiting 计时等待
- Terminated 终止

![生命周期](thread生命周期.jpg)

## 核心方法

- `void start()`启动
- `void run()`调用内部Runnable的`run()`。
- `public static void sleep(long millis)`休眠指定的毫秒数。
- `static void yield()`使当前运行的线程向另一个线程交出运行权。
- `getState()`获取当前线程状态。
- `void stop()` 终止线程，已废弃。
- `void suspend()` 暂停线程，已废弃。
- `void resume()` 恢复已暂停的线程，已废弃。
- `void interrupt()` **请求终止线程**，变更标记为，需要用`isInterrupted()`判断。还会抛出`InterruptedException`。
- `public static boolean interrupted()` 检查是否被中断，会重置中断状态为false。
- `boolean isInterrupted()` 作用同上，但是不会重置状态。
- `static Thread currentThread()`获取当前正在执行的线程对象。
- `void setName(String name)` 给线程起个名。
- `void setDaemon(boolean isDaemon)` 转为守护线程。

## 阻塞

- 当一个线程要访问对象时（参与竞争对象锁），而该对象锁被正在被其他线程持有，此线程就会阻塞，直到其他线程释放对象锁，并且线程调度器允许此线程持有对象锁。
- 当线程等待另一个线程通知调度器出现一个条件时，这个线程会进入等待状态。比如调用了`Object.wait()`或`Thread.join()`方法，或者是java.util.concurrent库中的`Lock.tryLock()`、`Condition.await`。

## 终止

- 任务执行完毕，run方法正常退出，线程正常终止。
- 抛出异常终止。
- 调用`stop()`方法。

## 中断

- `mThread.interrupt()`可中断线程，但并不会终止，而只是变更了中断状态。
- `Thread.currentThread().isInterrupted()`可以检查中断状态。
- 如果线程正在阻塞中，调用`interrupt()`方法会抛出`InterruptedException`异常。捕获这个异常后，可以在线程内部的catch里面在此调用`interrupt()`方法变更状态。

## 守护线程

守护线程是为其他线程提供服务的线程。JVM退出时会自动回收守护线程。

```java
mThread.setDaemon(true);
```

典型的就是GC线程。

## 未捕获异常处理器

线程的`run`方法不能包抛出任何检查型异常，但是非检查型异常会导致线程终止。

可以通过实现`Thread.UncaughtExceptionHandler`接口，自定义一个未捕获异常处理器：

```java
//创建线程
Thread t = new Thread();
//创建处理器
UncaughtExceptionHandler handler = new UncaughtExceptionHandler();
//设置处理器
t.setUncaughtExceptionHandler(handler);
//设置全局默认处理器
Thread.setDefaultUncaughtExceptionHandler(handler);


static class UncaughtExceptionHandler implements Thread.UncaughtExceptionHandler{

    @Override
    public void uncaughtException(@NonNull Thread t, @NonNull Throwable e) {
        //可以做异常采集、打印日志
    }
}
```

## 线程优先级

可以使用`void setPriority(int newPriority)`设置线程优先级，值在1~10，默认为5。系统自带的常量如下：

- MIN_PRIORITY = 1  最小优先级
- NORM_PRIORITY = 5 默认优先级
- MAX_PRIORITY = 10 最大优先级

默认情况下，线程会继承创建他自身的线程的优先级。

当线程调度器有机会选择新线程时，优先选择高优先级。

## 线程间通信

- wait()、notifyAll()
- Handler+Message
- 共享变量
- 信号量

