---
title: "_Index"
subtitle: ""
date: 2022-06-04T17:31:36+08:00
lastmod: 2022-06-04T17:31:36+08:00
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
最近入职了新公司，非常注重设计，开发之前要画大量的设计图，赞一个先，然后我需要回顾并学习多种图形的画法。
<!--more-->

## 选定工具

vscode + yuml插件

之所以选择这个方案，是因为vscode是目前我用的最多的编辑器，几乎什么都能编辑，而且有大量的插件可以辅助，非常灵活和高效，美中不足的是对kotlin支持不佳，而且自定义插件需要依赖node环境，我个人是非常不喜欢电脑上装一堆用不到的环境。

而yuml是我已知的非常轻量的环境，什么原理我是不知道的，但是它好在不需要依赖众多的环境，比如plantUml依赖java环境，dotUml依赖一个绘画库。。

## yuml插件安装

在vscode的插件市场搜索"yUML"插件安装即可。当使用vscode创建并打卡.yuml格式的文件后，可以在编辑器右侧打开预览窗口。

## yuml声明

我们在编写xml文件时，需要在文件起始位置用注释的形式做一个声明，如：

```xml
<?xml version="1.0" encoding="utf-8"?>
```
类似的，yuml文件也同样如此：

```
// {type:class}
// {direction:topDown}
// {generate:true}
```

**这里不要忘记"//"符号后面要加个空格**。

解释如下：

- type uml类型，包含：
  + class,类图
  + Activity，流程图；
  + Use-case，用例图；
  + State，状态图；
  + Deployment，发布图；
  + Package，包结构图；
  + Sequence，时序图。
- direction,绘制方向，默认从上到下
  + topdown，默认值，从上到下；
  + leftToRight，从左到右；
  + rightToLeft，从右到左。
- generate,是否生成svg图片：
  + true，在同目录生成一个同名的svg图；
  + false，默认值，不生成。