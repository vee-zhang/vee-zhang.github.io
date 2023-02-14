---
title: "Flutter的生命周期函数"
subtitle: ""
date: 2021-07-23T14:32:39+08:00
lastmod: 2021-07-23T14:32:39+08:00
draft: false
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

1. 构造函数
2. `initState` 初始化状态，只调用一次；
3. `didChangeDependencies`在initState执行完立刻调用，当依赖的InheritedWidget发生build时也会调用；
4. `build`画画用的；
5. `didUpdateWidget`状态改变时调用，如调用了`setState`函数；
6. `deactivate`相当于Android的onStop方法；
7. `dispose`销毁时调用。

## 测试

### 页面打开

```bash
I/flutter (30299): 调用了initState
I/flutter (30299): 调用了didChangeDependencies
I/flutter (30299): 调用了build
```

### 页面刷新

#### 状态没变

测试代码：

```dart
@override
Widget build(BuildContext context) {
  print('调用了build');
  return Scaffold(
    appBar: AppBar(),
    body: Container(
      child: MaterialButton(
          child: Text('主动刷新页面'),
          onPressed: () {
            print('我主动调用了setState');
            setState(() {});
          }),
    ),
  );
}
```

输出：

```bash
I/flutter (30299): 我主动调用了setState
I/flutter (30299): 调用了build
```

可以看到状态没有变化的时候，是不会调用`didUpdateWidget`函数的。

#### 状态改变测试1

测试代码：

```dart
///创建一个变量，通过改变这个变量来改变状态
int i = 0;

@override
Widget build(BuildContext context) {
  print('调用了build');
  return Scaffold(
    appBar: AppBar(),
    body: Container(
      child: MaterialButton(
          child: Text('主动刷新页面'),
          onPressed: () {
            print('我主动调用了setState');
            setState(() {
              i = 1;
            });
          }),
    ),
  );
}
```

输出：

```bash
I/flutter (30299): 调用了initState
I/flutter (30299): 调用了didChangeDependencies
I/flutter (30299): 调用了build
```

这里可以看到，我把赋值操作**写在setState的回调里面，可以触发`调用了didChangeDependencies`**。

#### 状态改变测试2

测试代码：

```dart
///创建一个变量，通过改变这个变量来改变状态
int i = 0;

@override
Widget build(BuildContext context) {
  print('调用了build');
  return Scaffold(
    appBar: AppBar(),
    body: Container(
      child: MaterialButton(
          child: Text('主动刷新页面'),
          onPressed: () {
            print('我主动调用了setState');
             i = 2;
            setState(() { });
          }),
    ),
  );
}
```

输出：

```bash
I/flutter (30299): 我主动调用了setState
I/flutter (30299): 调用了build
```

这里可以看到，我把赋值操作**写在setState的回调外面，`didChangeDependencies`函数就不会触发了。

#### 总结

当使用`setState`方法时：
- 赋值写在括号里面，可以触发`didChangeDependencies`函数；
- 赋值写在括号外面，不能触发`didChangeDependencies`。

### 页面退出

```bash
I/flutter (30299): 调用了deactivate
I/flutter (30299): 调用了dispose
```