---
title: "HTTPS"
subtitle: ""
date: 2022-06-16T10:58:50+08:00
lastmod: 2022-06-16T10:58:50+08:00
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

### HTTPS

通过在TCP层之上引入`SSL`或`TLS`层，对数据进行非对称加密，接收端通过公钥来解密，以验证对方身份以及数据完整性保护。主要用来防止中间人攻击。

由**对称加密**和**非对称加密**共同保障安全性。

#### 流程：

1. TCP三次握手；
2. 客户端生成一个随机数R1，结合自己支持的加密算法上报服务端443端口；
3. 服务端会保留R1，选择一个共同支持的算法，并生成随机数R2，连同颁发机构的数字证书（含公钥）下放给客户端；
4. 客户端通过内置的根证书验证数字证书的合法性（如果根证书信任该证书）；
5. 证书验证通过后就能解析出公钥，客户端生成R3，通过公钥非对称加密后上报服务端；
6. 双方通过约定的加密算法，汇总R1、R2、R3生成各自的key，开始对称加密通讯。
