---
title: "七层协议"
subtitle: ""
date: 2021-08-23T22:03:46+08:00
lastmod: 2021-08-23T22:03:46+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: [网络协议]

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

![](七层协议.jpeg)

#### 物理层

作用：定义一些电器、机械、过程和规范，如集线器；

设备：集线器。

#### 数据链路层

作用：负责把物理层的比特流封装成帧数据传递给上一层，也支持逆向。

设备：交换机通过MAC地址转发数据，逻辑链路控制。

#### 网络层

作用：定义一个逻辑的殉职，选择最佳路径传输，路由数据包。

典型协议：IP。

设备：路由器。

#### 传输层

作用：提供可靠和尽力而为的传输。

协议：TCP，UDP

负责网络传输和会话的建立。

#### 会话层

作用：控制会话，建立管理终止应用程序会话。

协议：SQL、JSP

负责会话建立；

#### 表示层

作用：格式化数据。

协议：json、png、wav

#### 应用层

网络传输的应用程序，如Http。
