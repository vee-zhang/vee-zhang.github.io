---
title: "Android事件分发机制"
subtitle: ""
date: 2022-03-11T16:20:55+08:00
lastmod: 2022-03-11T16:20:55+08:00
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

## 传递流程

1. Activity的`disPatchTouchEvent`方法接受到事件，然后传递到PhoneWindow的`disPatchTouchEvent`，再从PhoneWindow传递到DecorView的`disPatchTouchEvent`。
2. 然后事件会到达根ViewGroup的`disPatchTouchEvent`，在这里会首先调用`onInterceptTouchEvent`判断是否需要拦截。如果拦截，则调用自己的`onTouchEvent`处理；不拦截则会遍历子View进行处理。
3. 事件到达子View，会经`disPatchTouchEvent`到达`onTouchEvent`，如果进行了处理，就返回true，表示事件已经消费掉，不会继续传递。
4. 如果子View不处理并返回了false，事件会返回到父级ViewGroup的`disPatchTouchEvent`，然后直接到达ViewGroup的`onTouchEvent`。
5. 如果再不处理，就会继续向上冒泡，直到Activity的`onTouchEvent`。

## 需要注意

1. 只有ViewGroup有#；
2. 只有View的`onTouchEvent`是默认拦截的；
3. 重写`onTouchEvent`会出现警报，表示可能会引起click事件失效，所以需要手动调用`performClick`方法，指定click事件的触发时机。