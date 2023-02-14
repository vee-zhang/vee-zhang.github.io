---
title: "Dart-Future和Stream"
subtitle: ""
date: 2021-07-21T17:52:47+08:00
lastmod: 2021-07-21T17:52:47+08:00
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

## Future 未来的一次回调

Future位于`event queue`事件队列中。

Future表示**未来的一次性**的任务回调。

### wait方法

类似RxJava中的`zip`操作符：

```dart
final futures = [
  Future.delayed(Duration(seconds: 3)),
  Future.delayed(Duration(seconds: 5))
];
Future.wait(futures)
    .then((value) => print('拿到值：$value'));
```
## Stream 持续监听

Stream是持续性监听，所以就必须要在不需要的时候手动释放。

```dart
final stream = File('').openRead();
//订阅
final listener = stream.listen((event) {
  //启动监听
});

//转换成Future
Future future = listener.asFuture();

//暂停
listener.pause();

//暂停恢复
listener.resume();

//取消
listener.cancel();
```

### Stream的订阅与广播

#### 订阅

Stream默认就是单订阅模式。如果重复调用`listen()`方法就会报错，比如：

```dart
final stream = File('').openRead();
//订阅
final listener = stream.listen((event) {
  //启动监听
});
final listener1 = stream.listen((event) {});
```

报错：

```bash

```

#### 广播模式

广播模式允许多订阅。可以把订阅模式转为广播模式：

```dart
final stream = File('').openRead();

stream.asBroadcastStream();
//订阅
final listener = stream.listen((event) {
  //启动监听
});
final listener1 = stream.listen((event) {});
```

也可以通过`StreamController`直接创建广播：

```dart
final streamController = StreamController.broadcast();
streamController.stream.listen((event) {
  // todo 回调
});
streamController.add('value');
streamController.close();
```

#### StreamSink

