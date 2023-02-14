---
title: "Crash监控"
subtitle: ""
date: 2021-08-09T10:17:20+08:00
lastmod: 2021-08-09T10:17:20+08:00
draft: false
authors: []
description: ""

tags: [Android, 性能优化]
categories: []
series: [Android性能优化]

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
Crash（应用崩溃）是由于代码异常而导致 App 非正常退出，导致应用程序无法继续使用，所有工作都停止的现象。发生 Crash 后需要重新启动应用（有些情况会自动重启），而且不管应用在开发阶段做得多么优秀，也无法避免 Crash 发生，特别是在 Android 系统中，系统碎片化严重、各 ROM 之间的差异，甚至系统Bug，都可能会导致Crash的发生。在 Android 应用中发生的 Crash 有两种类型，Java 层的 Crash 和 Native 层 Crash。这两种Crash 的监控和获取堆栈信息有所不同。

<!--more-->

### 什么是崩溃

未处理的**异常**或**信号**导致的意外退出，会使 Android 应用崩溃。

当应用崩溃时，Android 会终止应用的进程并显示一个对话框，告知用户应用已停止。

使用 Java 编写的应用会在抛出未处理的异常（由 Throwable 类表示）时崩溃。使用原生代码语言编写的应用，会在执行过程中遇到未处理的信号（如 SIGSEGV）时崩溃。

### Java Crash 监控

我们可以通过`Thread.setDefaultUncaughtExceptionHandler`方法来设置线程的默认异常处理器。

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        Thread.setDefaultUncaughtExceptionHandler(new MyUncaughtExcetionHandler());
    }


    public static class MyUncaughtExcetionHandler implements Thread.UncaughtExceptionHandler {
        @Override
        public void uncaughtException(@NonNull Thread t, @NonNull Throwable e) {

        }
    }
}
```

Thread的默认UncaughtExceptionHandler，是`RuntimeInit#KillApplicationHandler`，功能就是**杀掉程序**，所以我们不更改的话，程序就会弹窗并退出。

#### 测试行为

测试方式，我在UI里面定义了一个按钮，点击后会触发一个未捕获的NPE：

```java
public void test(View v) {
    String content = null;
    content.length();
}
```

实测点击程序崩溃。

当设置了`Thread.setDefaultUncaughtExceptionHandler`之后，再次点击按钮触发NPE，**可以拦截到未捕获异常，而且程序没有崩溃**。



### NDK Crash 监控

#### Linux信号机制

**linux信号机制是IPC的一种方式，同时也负责监控系统异常及中断**。

当程序发生运行异常时，linux内核将会产生错误信号并通知当前进程，当前进程在收到该错误信号后，可以又三种处理方式：

- 忽略信号；
- 捕捉信号并执行信号处理逻辑；
- 执行信号的默认操作（记录信息，然后终止进程）。

#### crash信号

|信号|描述|
|---|---|
|SIGSEGV|内存引用无效|
|SIGBUS|访问内存对象的未定义部分|
|SIGFPE|算术运算错误（0做除数）|
|SIGILL|非法指令，如执行垃圾或特权指令|
|SIGSYS|糟糕的系统调用|
|SIGXCPU|超过CPU时限|
|SIGFSZ|超出文件大小限制|

一般的出现崩溃信号，Android系统默认缺省操作是直接退出我们的程序。但是**系统允许我们给某一个进程的某一个特定信号注册一个相应的处理函数（signal），即对该信号的默认处理动作进行修改**。因此NDK Crash的监控可以采用这种信号机制，捕获崩溃信号执行我们自己的信号处理函数从而捕获NDK Crash。

### 墓碑文件

Android本机程序本质上就是一个Linux程序，当它在执行时发生严重错误，也会导致程序崩溃，然后产
生一个记录崩溃的现场信息的文件，而这个文件在Android系统中就是 tombstones 墓碑文件。位于`~/data/tombstones/`下。**普通应用没有读取权限**。

