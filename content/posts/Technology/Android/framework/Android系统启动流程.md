---
title: "Android系统启动流程"
subtitle: ""
date: 2021-07-28T16:15:50+08:00
lastmod: 2021-07-28T16:15:50+08:00
draft: false
authors: []
description: "xxxx什么垃圾玩意儿，看的头疼"

tags: [Android]
categories: []
series: [AndroidFramework]

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

### 概述

1. 启动电源时，芯片先从固定位置加载`BootLoader`并执行；
2. BootLoader拉起linux内核；
3. linux内核启动后加载`init.rc`并启动init进程；
4. init进程中初始化并启动**属性服务**并启动`Zygote`进程；
5. Zygote负责创建JVM，注册JNI，创建**服务端Socket**，最后启动`SystemServer`；
6. SS中创建`Binder线程池`，并启动系统服务，包括`AMS`和`PMS`；
7. PMS负责安装APK，AMS则启动Launcher。

### 内核空间流程

1. 按下开机键。
2. 加载启动引导程序bootloader到内存并执行。
3. 拉起linux内核，设置缓存，加载驱动.

### 用户空间流程

1. 内核启动后，启动init进程(pid=1)调用init的入口main函数。
2. 创建和挂载文件目录。
3. 初始化并启动【属性服务】。
4. 注册子进程信号处理函数，用来处理进程终止信号，防止僵尸进程。
5. 解析init.rc文件，并启动Zygote。
6. 根据不同系统读取不同的init。zygotexx.rc启动zygote进程（孵化器）。
7. 启动SystemServer
8. 创建launcher

### Zygote启动流程

1. 创建JVM;
2. 通过JNI调用`ZygoteInit`这个类的`main`函数，开始**进入java框架层*(´▽`ʃ♡ƪ)；
3. 创建一个Socket服务端；
4. fork出SystemServer进程（该进程中启动系统服务）；
5. 调用`zygote.runSelectLoop`死循环，开始监听AMS的请求。

### SystemServer的处理过程

`SystemServer` 进程用来创建系统服务，如AMS、WMS和PMS等，他的处理过程为：

1. 启动Binder线程池；
2. 创建`SystemServerManager`，用于对系统服务创建、启动和生命周期管理；
3. 启动系统服务。

这里提取一个关键点：**`SystemServerManager`是用来管理系统服务的**。

### Launcher启动过程

1. SystemServer启动时会同时启动`AMS`和`PackageManagerService`；
2. 而PMS启动后会安装APK；
3. AMS会调起Launcher；
4. Launcher在`onCreate`中渲染应用图标；