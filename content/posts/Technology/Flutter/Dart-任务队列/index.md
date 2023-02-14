---
title: "Dart-任务队列"
subtitle: ""
date: 2021-07-21T17:23:36+08:00
lastmod: 2021-07-21T17:23:36+08:00
draft: false
authors: []
description: ""

tags: [Dart]
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

## Dart的消息队列机制

Dart像Android一样，是基于消息驱动的.

依靠`eventLoop`不停的从队列中获取消息或者事件来驱动整个应用的运行，包括isolate发送过来的消息就是通过Loop处理。

看起来很像Android中的`handler+looper+msg`机制。

但是不同的是，Android中每个线程只有一个looper对应一个一个MessageQueue，但是dart中存在**两个队列**：
  - `event queue`事件队列；
  - `microtask queue`微服务队列。

## Dart的消息传递过程

![](images/../../../static/images/dart消息机制.png)

Dart在执行完`main`方法后，就会由loop开始执行两个任务队列中event。

loop会先执行完所有的微服务队列中的所有event，再去执行事件队列中的一个事件，one by one，所以**微服务队列的优先级高于事件队列**，我们可以利用这项特性来插队：

eg:打印文件内容

```dart
import 'dart:io';

void main(){
  new File("/Users/enjoy/a.txt").readAsString().then((content){
      print(content);
  });

  //死循环造成阻塞
  while(true){}
}
```

文件内容永远也无法打印出来，因为main函数还没执行完。而then方法是由Loop检查Event queue执行的。

如果需要往微服务中插入Event进行插队：

```dart
import 'dart:async';
import 'dart:io';
//结果是先执行了microtask然后执行then方法。
void main(){
  new File("/Users/enjoy/a.txt").readAsString().then((content){
      print(content);
  });
  //future内部就是调用了 scheduleMicrotask
  Future.microtask((){
    print("future: excute microtask");

  });
//  scheduleMicrotask((){
//    print("");
//  });

}
```

>注意：如果微服务中出现阻塞，那么两个队列中所有的后续任务都不会执行，也是因为单线程模型导致。