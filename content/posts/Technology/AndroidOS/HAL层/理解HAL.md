---
title: "理解HAL"
subtitle: ""
date: 2022-07-18T11:53:55+08:00
lastmod: 2022-07-18T11:53:55+08:00
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

<!--more-->

## HAL的作用

连通硬件驱动，提供一个标准接口（硬件供应商自己提供实现），让Android可以忽略底层驱动程序的实现。

## HAL类型

- 绑定式：以HAL接口定义语言或者AIDL表示的HAL。`FrameWork`与`HAL`通过`Binder`进行IPC通信，自Android8.0之后的版本都必须支持绑定式。
- 直通式：以HIDL封装的传统HAL或者[旧版HAL](https://source.android.google.cn/devices/architecture/hal?hl=zh-cn)。

## HAL接口定义语言HIDL

类似AIDL,安卓提供了一种用于HAL和其用户之间的接口的表述语言——HIDL。

>当然非常google的是，HIDL在10.0废弃了，直接用AIDL替换了他，也就是说现在玩9.0车机的还是要学这玩意，然后很快就白学了。

HIDL也是用于描述IPC的，底层仍然采用`Binder`。他被设计出来的目的就是**可以在无需重新构建HAL的情况下替换框架**（简言之就是一种用于动态化实现的描述）。

HIDL是由供应商（比如我们长城）或者SOC供应商（高通）构建出来，放在`/verdor`分区（供应商分区）内。