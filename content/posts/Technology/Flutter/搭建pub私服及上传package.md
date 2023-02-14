---
title: "搭建pub私服及上传package"
subtitle: ""
date: 2021-07-15T14:13:55+08:00
lastmod: 2021-07-15T14:13:55+08:00
draft: false
authors: []
description: ""

tags: [flutter]
categories: [technology]
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
事情的起因是，在工作项目中，一开始只有我一个人研发，为了方便，我封装了一个网络访问层。但是随着团队规模的拓展，陆续加入了其他人，时间紧项目重，所以后续的伙伴没有时间来问我这个框架怎么使用，所以他们直接上手改了我的封装！但是后期架构要求加入oauth2.0机制，所以需要全局处理token的有效认证，并且自动刷新token。为了满足这一需求，我需要重新编写网络层，同时为了避免伙伴修改我的抽象，我想到了本文的主题——搭建个pub.dev私服吧！
<!--more-->

## server配置

首先到我的[码云](https://gitee.com/william198824/pub_server)clone个项目下来,然后习惯性`flutter pub get`。

接下来修改配置，修改`pub_server/example/src/example.dart`文件,找到`argsParser()`：

```dart
ArgParser argsParser() {
  var parser = ArgParser();

  parser.addOption('directory',
      abbr: 'd', defaultsTo: 'pub_server-repository-data');

    //host地址
  parser.addOption('host', abbr: 'h', defaultsTo: 'localhost');

    //端口号
  parser.addOption('port', abbr: 'p', defaultsTo: '8080');

  // 运行模式
  parser.addFlag('standalone', abbr: 's', defaultsTo: false);
  return parser;
}
```

由于我的8080端口已经被jenkies占用了，所以只能把pub的端口改为9090。要注意的是，**host默认是localhost，只支持本机访问**，如果我们要挂到服务上，需要把这里改为`0.0.0.0`之后，我们才能通过ip或者域名访问服务。


然后启动服务：

```shell
dart example/example.dart -d /tmp/package-db
```

>如果出现「To make the pub client use this repository configure...」表示服务启动成功！

## 创建测试package

怎么创建，不说了，这里只贴出yaml文件：

```yaml
name: lk_dio
description: 用于来康科技公司的网络请求层封装，包括平台、大脑的接口调用规则和token有效期验证及自动刷新机制。
version: 0.0.2
author: William <自己的邮箱@enn.cn>
homepage: 'http://项目主页地址.com'
publish_to: 'http://localhost:9090'


environment:
  sdk: ">=2.7.0 <3.0.0"
  flutter: ">=1.17.0"

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
```

配置好之后,可以在本地直接依赖：

```yaml
lk_dio:
    path: user/william/lk_dio
```

当然也可以发布到我们的pub私服上，发布之前可以通过命令检查错误：

```shell
flutter packages pub publish --dry-run
```

按照提示解决问题，然后发布：

```shell
flutter packages pub publish
```

出现如下信息表明发布成功：

```shell
|-- lib
|   '-- helloworld.dart
|-- pubspec.yaml
'-- test
    '-- helloworld_test.dart

Looks great! Are you ready to upload your package (y/n)? y
Uploading...
Successfully uploaded package.
```

但是如果不FQ，是一定不会成功的，你看到的将是如下信息：

```shell
Pub needs your authorization to upload packages on your behalf.
```

失败的原因就是需要google的认证，怎么办，fq? 有没有更好的办法？

## 绕过google认证

再clone[这个项目](https://gitee.com/william198824/pub)之后`flutter pub get`，然后执行：

```shell
dart --snapshot=mypub.dart.snapshot bin/pub.dart 
```

完事后会自动生成一个`mypub.dart.snapshot`。

复制之后放入${flutterSDK Path}/bin/cache/dart-sdk/bin/snapshots/ 目录下

用txt编辑器打开${flutterSDK Path}/bin/cache/dart-sdk/bin/pub文件，将倒数第三行的：`pub.dart.snapshot` 替换为 `mypub.dart.snapshot`,然后重新发布package就OK了。

## 依赖自己的package

```yaml
lk_dio:#这里要与之前一致
  hosted:
    name: lk_dio #这里要与之前一致
    url: http://localhost:9090
  version: ^1.0.0
```

添加了依赖之后，我`flutter pub get`，本机没问题，项目正常跑，万分激动，但是。。。

当我把server发布到公司服务器后，**publish失败！**经查，是运维没有开放9090端口，找过运维之后问题解决。

然后我再添加依赖，运行pub get，竟然卡住不动了，内心瞬间一万只草泥马德，上传可以下载就不行怪了！后来发现server的配置文件中有个配置：

```dart
// 运行模式
  parser.addFlag('standalone', abbr: 's', defaultsTo: false);
```

`standalone`好像是独立部署的意思。

把这里的`defaultTo`的值改为`true`,重新部署、启动，再重新下载依赖pub get，等等足足71秒后竟然成功了！后来运维解释，之所以这么慢是因为从北京访问我们盐城的服务器，而且没有CDN加速。

## 鸣谢：

https://www.jianshu.com/p/59f4778864f0