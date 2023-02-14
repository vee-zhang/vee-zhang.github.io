---
title: "OOM与内存优化"
subtitle: ""
date: 2021-08-03T10:16:07+08:00
lastmod: 2021-08-03T10:16:07+08:00
draft: false
authors: []
description: ""

tags: [Android,性能优化]
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

### Java对象的生命周期

1. 创建
    为对象分配内存空间，从父到子依次调用构造函数，构造对象。
2. 应用
    对象至少被一个强引用持有。
3. 不可见
    当没有强引用时，对象不可见。
4. 不可达
    GC开始做可达性分析。
5. 收集
    对象被标记不可达对象，等待GC回收。
6. 终结
   回收对象，重新分配内存空间。

### Java对象的内存布局

- 对象头
- 实例数据
- 对齐填充（非必须）