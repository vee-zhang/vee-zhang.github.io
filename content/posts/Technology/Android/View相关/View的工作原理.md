---
title: View的工作原理
date: 2021-03-22 10:37:36
tags: [Android, View]
---

## 三大流程

- measure 用来测量View的宽高；
- layout    用来确定View在父容器中的放置位置；
- draw  负责绘制。


在ActivityThread中，当Activity创建完毕后，会将`DecorView`附加到`Window`中，同时会创建`ViewRootImpl`对象，并将`ViewRootImpl`与`DecorView`建立关联。

ViewRootIml其实是`DecorView`的管理类。

View的绘制流程是从ViewRoot的`performTraversals()`方法开始的，经过`measure`、`layout`、`draw`三个过程才能最终将一个View绘制出来。

### measure

通过调用`setMeasuredDimension`决定View的宽高。

计算之后可以通过`getMeasureWidth`和`getMeasureHeight`获取测量后的结构。

### measureChildren

通过遍历调用`measureChild(child,widthMeasureSpec,heightMeasureSpec)`方法完成所有子View的测量。

### layout

用来确定子元素的位置。需要遍历子元素，调用其`layout`方法。

决定了View的四个顶点的坐标和实际的View的宽和高。

- getTop
- getLeft
- getRight
- getBottom
- getWidth
- getHeight

`getWidth`和`getHeight`才是View的最终宽高。

## onDraw

1. 绘制背景background.draw(canvas)。
2. 绘制自己（onDraw）。
3. 绘制children（dispatchDraw）。
4. 绘制装饰（onDrawScrollBars）。

## DecorView

DecorView作为顶级的View，包含一个竖直的`LinnerLayout`，内部含有**标题栏和内容栏**.`setContentView`就是把View附加到内容栏。内容栏是个`FrameLayout`，他的的id是`content`，所以是setContentView。

```java
//获取content内容栏
ViewGroup content= findViewById (R.android.id.content)；

//获取我们自己的顶级View
View rootView = content.getChildAt(0);
```

## MeasureSpec

这是一个32位的int，高2位代表SpecMode，低30位代表SpecSize。

通过将SpecMode和SpecSize打包成一个int，避免过多的内存分配。

- UNSPECIFIED 父容器不做任何闲置，子View可以任意大；
- EXACTLY 父容器测出View所需大小，对应match_parent和精确的值；
- AT_MOST   父容器制定了一个可用大小SpecSize，View的大小不能大于这个值，用于wrap_content。

## 获取View的宽高

view的measure过程和Activitu生命周期不同步，所有在onCreate、onStart、onResume中都不能获取到View的宽高。

- Activity/View#onWindowFocusChanged；
- view.post(runnable)；
- ViewtreeObserver
- view.measure(int widthMeasureSpec,int heightMeasureSpec)(手动测量);


## 生命周期

1. `Constructors`构造函数；
2. `onFinishInflate`当该View及其子View从XML文件中填充完成后会被调用；
3. `onAttachedToWindow`附加到窗口;
4. `onMeasure`计算尺寸时调用；
5. `onSizeChanged`当前view尺寸变化时调用;
6. `onLayout`调用所有子view的`layout`方法为每一个子view确定位置；
7. `onDraw`绘图时调用；
8. `onDetackedFromWindow`脱离窗口时调用。