---
title: "ANR问题解决"
subtitle: ""
date: 2021-08-04T13:39:14+08:00
lastmod: 2021-08-04T13:39:14+08:00
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

<!--more-->

### ANR的概念

是指应用程序未响应。

由 `ActivityManager` 和 `WindowManager` 两个系统服务进行监视的。

### ANR产生场景

1. 主线程耗时操作
2. SP
3. 序列化
4. 并发造成死锁，block了主线程
5. 系统资源耗尽

### 行为

* 前台产生ANR弹框询问用户是否等待；
* 后台产生的ANR直接崩溃。

### 时限

| 场景            | 时常 |
| --------------- | ---- |
| 前台服务        | 20S  |
| 后台服务        | 200S |
| 前台广播        | 10S  |
| 后台广播        | 60S  |
| ContentProvider | 10S  |
| 输入事件        | 5S   |

### ANR的错误日志

```shell
2021-08-04 14:07:55.885 4563-4574/com.vee.myapplication I/e.myapplicatio: Thread[7,tid=4574,WaitingInMainSignalCatcherLoop,Thread*=0x7250d39c00,peer=0x167402a0,"Signal Catcher"]: reacting to signal 3
2021-08-04 14:07:56.015 4563-4574/com.vee.myapplication I/e.myapplicatio: Wrote stack traces to tombstoned
2021-08-04 14:08:07.027 4563-4563/com.vee.myapplication I/Choreographer: Skipped 1202 frames!  The application may be doing too much work on its main thread.
```

#### 最重要的一句话

**The application may be doing too much work on its main thread.**

### ~~trace文件~~

位置： `data/anr/traces_*.txt`

找到firstPid，即发生ANR的进程id。

trace包含ANR的时间点和CPU使用率、主线程状态、其他线程状态等重要信息。

但是5.0之后会被selinux挡住，获取不到。

### 墓碑文件

看log中间的一条 `Wrote stack traces to tombstoned` ，意思已经把对战追踪写入了墓碑文件。**墓碑文件没有读写权限**。

### 线上监控

线上监控的难点就在于，前台发生的ANR没有崩溃

#### ~~FileObserver~~

此方案基本不能用了。

原理是监控 `/data/anr` 文件夹的变化，然后判断pid。

#### ~~BlockCanary~~

由于Android是基于消息驱动的，在Loop.loop()方法中：

```JAVA
public static void loop(){
  ...
  for(;;){
    Message msg = queue.next();

    if(logging != null){
      logging.print(">>>>Dispatching to"+msg.target+msg.what);
    }

    msg.target.dispatchMessage(msg);
  }
}
```

根据 `queue.next()` 与 `msg.target.dispatchMessage(msg)` 的时间差来判断是否发生ANR，做法是自定义一个looging传进去：

```java
Looper.getMainLooper.setMessageLogging(myLogging);
```

通过重写 `print` 方法加入我们自己的逻辑。

问题：logging有可能被改成null，反而造成NPE问题。

#### ~~SafeLooper~~

反射hook主线程的looper，接管功能，插入时间差判断，但反射导致大量性能损耗。

#### ~~仿WatchDog~~

启动一个守护线程，先向主线程发送一个消息，再通过 `handler.postDelay()` 休眠，而delay时间就是ANR的阈值，delay到期后判断主线程是否确实收到消息。

#### 微信的解决方案

[腾讯Matrix for Android对ANR监控的原理](https://cloud.tencent.com/developer/article/1848945)

> 看完这篇文章，让我深深的认识到了什么是差距，什么是价值。
