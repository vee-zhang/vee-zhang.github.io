---
title: "Flutter 模块、包和插件(待续...)"
subtitle: ""
date: 2021-07-23T16:23:52+08:00
lastmod: 2021-07-23T16:23:52+08:00
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

## 三者之间的区别

- Flutter Module：用来把flutter功能作为组件集成到Android或者iOS主项目中；
- Dart package：纯Dart工程，不包含Platforms，就是一个可以外置的包，里面存放各种类文件;
- Plugin packages：类似于主工程的插件，可以包含各种Platforms和本土代码。

>这个是官方文档上面理所当然的解释，但是事实上，package和plugin就是同一个玩意：包里配置上platforms工程，就成了plugin；而相对的plugin去除掉flatforms之后，那就成了包。但是为啥要区分开呢，因为官方针对同一个东西给出了两种模板而已。

## 创建工程

```sh
# 创建包
flutter create -t plugin --platforms android --org com.vee  plugin_test
```

采用脚本创建的话，**必须要自己指明包含的平台**，否则不会创建Android或者iOS的本土代码。比如以上代码，我通过`platforms`选项，只选择了Android平台，所以flutter生成的项目中也只含有安卓的包，而没有iOS的包。

## 通信

三种通道：
  - BasicMessageChannel：用于传递字符串和半结构化的信息；
  - MethodChannel：用于传递方法调用（method invocation）；
  - EventChannel: 用于数据流（event streams）的通信。

PS：消息编解码器，是JSON格式的二进制序列化，所以调用方法的参数类型必须是可JSON序列化的。 PS：方法调用，也可以反向发送调用消息。