---
title: "Flutter支持单向滚动的PageView"
subtitle: ""
date: 2021-07-21T11:42:03+08:00
lastmod: 2021-07-21T11:42:03+08:00
draft: true
authors: []
description: ""

tags: [Flutter]
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

## 代码

```dart
import 'package:flutter/material.dart';

/// A controller for [ControlablePageView]，extends [PageController].
class ControllablePageController extends PageController {
  ControllablePageController({
    this.actionDistance = 200,
    this.canNext = true,
    this.canPrevious = true,
    int initialPage = 0,
    bool keepPage = true,
    double viewportFraction = 1.0,
  }) : super(initialPage: initialPage);
  bool canNext;
  bool canPrevious;
  int actionDistance;
}

/// A container contains [PageView], support single direction drag slide.
/// 问题：目前滑动不跟手，不好解决
class ControlablePageView extends StatelessWidget {
  ControlablePageView(this._itemBuilder, this._itemCount,
      {ControllablePageController pageController, this.onPageChanged}) {
    this._controller = pageController ?? ControllablePageController();
  }

  ControllablePageController _controller;

  num downX = 0;
  bool isAni = false;

  double _currentDistance;

  IndexedWidgetBuilder _itemBuilder;
  int _itemCount;

  ValueChanged<int> onPageChanged;

  @override
  Widget build(BuildContext context) => Listener(
        child: PageView.builder(
          physics: NeverScrollableScrollPhysics(),
          itemCount: _itemCount,
          controller: _controller,
          itemBuilder: _itemBuilder,
          onPageChanged: onPageChanged,
        ),
        onPointerDown: (PointerDownEvent detail) {
          downX = detail.position.dx;
          isAni = true;
        },
        onPointerMove: (PointerMoveEvent detail) {
          if (!isAni) return;
          //_currentDistance>0手指右滑，反之则为左滑
          _currentDistance = detail.position.dx - downX;
          if (_currentDistance.abs() < _controller.actionDistance) return;
          if (_currentDistance > 0 && _controller.canPrevious) {
            _controller.previousPage(
                duration: Duration(milliseconds: 500), curve: Curves.easeInOut);
          } else if (_currentDistance < 0 && _controller.canNext) {
            _controller.nextPage(
                duration: Duration(milliseconds: 500), curve: Curves.easeInOut);
          }
          isAni = false;
        },
        onPointerUp: (detail) => isAni = false,
      );
}
```

## 调用

```dart
import 'package:flutter/material.dart';
import 'package:left_scroll_test/ControlablePageView.dart';
import 'package:left_scroll_test/home.dart';

class HomeRoute extends StatefulWidget {
  @override
  State<StatefulWidget> createState() => _HomeRouteState();
}

class _HomeRouteState extends State<HomeRoute> {
  final _cntroller = ControllablePageController();

  bool canLeft = true;
  bool canRight = true;

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(
          actions: [
            Text("右划"),
            Switch(
              value: canRight,
              onChanged: (v) {
                setState(() {
                  canRight = v;
                  _cntroller.canPrevious = v;
                });
              },
            ),
            Text("左划"),
            Switch(
              value: canLeft,
              onChanged: (v) {
                setState(() {
                  canLeft = v;
                  _cntroller.canNext = v;
                });
              },
            ),
          ],
        ),
        body: ControlablePageView(
          (content, index) => Container(
            color: colors[index],
          ),
          colors.length,
          pageController: _cntroller,
        ),
      );
}
```

## 原理

首先设置**PageView不响应用户手势**，然后用`Listener`监听PageView的手势事件，依据事件的触发点与原点的距离得出方向，使进一步控制成为可能。然后通过`PageController`主动跳转到前、后页面。

## 不足

这种实现方式不可避免造成拖拽不跟手问题。我尝试使用`AbsorbPointer`拦截手势，使禁止方向的手势不会透传到PageView。但遇到了如下问题：

1. 根据第一个move事件来判定方向后，后面的事件不能透传到PageView。
2. 根据第一个move事件来判定方向，用户在后续的move操作中换了方向（手指触屏来回拖拽），导致每个move事件都要做方向判定，造成频繁调用`setState`重绘整个AbsorbPointer和内置的PageView。