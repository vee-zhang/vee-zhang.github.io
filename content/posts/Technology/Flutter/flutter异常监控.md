---
title: "Flutter异常监控"
subtitle: ""
date: 2022-04-20T14:06:58+08:00
lastmod: 2022-04-20T14:06:58+08:00
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
## 异常的捕获

可以使用try-catch捕获同步异常，使用catchERROR捕获异步代码的异常。

## 未捕获异常

可以在main函数中重写`FlutterError.onError`函数设置默认的未捕获异常处理机制。

```dart
void main(){
  Flutter.Error.onError = (detail){
    // 异常处理
  };
  
  runApp(MyApp());
}
```

## 捕获Future的异常

可以使用zone,zone表示一个代码执行的上下文，给异步代码提供了一个稳定的运行环境，可以理解为一个沙盒：

```dart
void main(){
  runZone((){
    runApp(MyApp());
  },onError(error,stackTrace){
    // 异常处理
  });
}
```

## 红白屏捕获

可以在main函数中重写ErrorWidgetBuilder：

```dart
void performRebuild() {
    ...
    try {
      ...
    } catch (e, stack) {
      // ErrorWidget.builder回调方法替换错误页面
      built = ErrorWidget.builder(
        _debugReportException(
          ErrorDescription('building $this'),
          e,
          stack,
          informationCollector: () sync* {
            yield DiagnosticsDebugCreator(DebugCreator(this));
          },
        ),
      );
    }
    ...
  }
  //默认的处理方式,我们也可以在main中覆盖处理
  static ErrorWidgetBuilder builder = _defaultErrorWidgetBuilder;
```