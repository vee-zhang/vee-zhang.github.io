---
title: "大数据量取top"
subtitle: ""
date: 2022-05-12T22:28:56+08:00
lastmod: 2022-05-12T22:28:56+08:00
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
一个无限大的文件，里面包函大量的数据，需求是从中取出top50。
<!--more-->

## 先说点废话吧

这是现在面试问的TM的最多的问题，解答时往往还不让用ide。我就在想现在都卷成什么样了，这种量级的数据怎么能拿到移动端来做呢，者面试题明显是从服务端抄过来的，用来考核移动端的能力有什么用？问这种问题的面试官用意也很明显，其目的根本不在于考察面试者的实际业务水平，而是仅仅为了看你有没有刷过题，比如玩过leetCode，所以他们最后招到的最多也就是理论家，这种人我都不信他有什么解决问题的能力。

## 解答方案

### 方案一：全排序

如果不考虑内存的话，可以采用快速排序法对整个文件做全排序，然后取top50。

```java

```

时间复杂度：O(n)

### 方案二：