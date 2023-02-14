---
title: "UI卡顿与布局优化"
subtitle: ""
date: 2021-08-04T10:39:30+08:00
lastmod: 2021-08-04T10:39:30+08:00
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

### Systrace

Systrace 是Android平台提供的一款工具，用于记录短期内的设备活动。该工具会生成一份报告，其中汇总了Android 内核中的数据，例如 CPU 调度程序、磁盘活动和应用线程。Systrace主要用来分析绘制性能方面的问题。在发生卡顿时，通过这份报告可以知道当前整个系统所处的状态，从而帮助开发者更直观的分析系统瓶颈，改进性能。

需要安装Python，好吧，滚！

### 过度渲染

#### 布局层级优化

开发者工具中开启**页面过度绘制监控**，在屏幕上用颜色冷暖色调来监控绘制次数。

使用**Layout Inspector**工具可以直观的检查布局层级。

- ConstraintLayout减少嵌套；
- merge标签重用父级布局；
- ViewStub懒加载不需要立即显示的视图元素。

#### 背景优化

去除不必要的背景，比如部分Activity、fragment中存在背景图片，但是并不可见，就需要移除。

#### 减少alpha的使用

对于不透明的View，只需要绘制一次即可，但是如果这个View设置了`Alpha`，则至少需要两次绘制。

解决方法是减少透明View的使用，或者直接使用ARGB替代。

#### 复杂页面异步处理

需要用到AsyncLayoutInflater类：

```groovy
dependencies {
implementation "androidx.asynclayoutinflater:asynclayoutinflater:1.0.0"
}
```

使用：

```java
new AsyncLayoutInflater(this)
  .inflate(R.layout.activity_main, null, new AsyncLayoutInflater.OnInflateFinishedListener() {
  @Override
  public void onInflateFinished(@NonNull View view, int resid, @Nullable ViewGroup parent) {
    setContentView(view);
    //......
  }
});

```

#### GC优化

由于Java的回收机制是stop the world，所以一旦发生GC就会挂起所有的线程，包括主线程，这就造成了UI的卡顿。为避免这种情况，需要尽可能少的发生GC，那么就要禁止在循环或者递归中创建无用的临时变量。

### 大招

系统在渲染页面前做了三件事：

1. 读取layout.xml
2. 递归解析xml
3. 层层反射生成对象

解决方式：

利用编译时注解处理器，通过java文件生成器（JavaPoet）在编译时生成布局的ViewGroup类，注入到Activity中。

缺点：不能使用merge标签，因为只有在运行时才知道父级使用什么布局。

### 自定义View优化

1. Draw方法中尽量不要创建对象。

