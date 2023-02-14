---
title: "单向链找倒数第N个节点"
subtitle: ""
date: 2022-03-24T20:26:52+08:00
lastmod: 2022-03-24T20:26:52+08:00
draft: false
authors: []
description: ""

tags: []
categories: [趣味题]
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
假设有个单向链，我们并不知道长度，要求找到倒数的第N个节点。
<!--more-->

## 解题思路

用常规的数学思路去解是不行的，我们可以用逻辑思路去解。

首先，既然是单向链，肯定有头有尾，就像一个轨道一样有起有终。那么假设我们有一列火车的长度就是N，当火车驶到终点时，车位的位置不就是我们要找的位置吗？

## 代码实操

```java
/**
  * 有个不知道长度的单向链node，我们要找到其倒数第num个节点返回
  * @param num
  * @return
  */
private static Node findNode(int num) {
    //定义头尾指针
    Node nodeHead = node;
    Node nodeTail = node;

    //让头先走num步
    for (int i = 0; i < num - 1; i++) {
        nodeHead = nodeHead.next;
    }

    //头尾一起，一步一个脚印，直到头触底
    while (nodeHead.next != null) {
        nodeTail = nodeTail.next;
        nodeHead = nodeHead.next;
    }

    //返回尾，此时尾指针指向的就是我们要找的num节点
    return nodeTail;
}
```