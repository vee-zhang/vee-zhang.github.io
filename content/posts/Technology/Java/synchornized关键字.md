---
title: synchornized关键字
date: 2021-01-06 16:57:01
tags: [java]
categories: [technology]
description: 
---

## 反编译指令

```
javap -v XXX.class
```

## 监视器对象Monitor

## 原理

当synchronized加载代码块上，JVM会执行两条指令：

- 加锁：MonitorEnter
- 解锁：MonitorExit

当synchornized关键字加在方法上，jvm会给该方法加一个flag：`ACC_SYNCHRONIZED`

## Java对象头

当我们在代码中new一个object，每个对象在内存中都会存放相关的数据，除了我们自己定义的属性之外，还存在着一个对象头，对象头存放着GC年龄（MarkWork），

## Synchornized中的锁分类

### 轻量锁

通过CAS操作来加锁和解锁，线程如果拿不到锁，会自旋，而不需要挂起线程，减少了上下文切换，相对性能较高。

### 自旋锁

不挂起线程，bu