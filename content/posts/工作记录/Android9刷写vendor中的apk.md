---
title: "Android9刷写vendor中的apk"
subtitle: ""
date: 2022-06-17T20:44:32+08:00
lastmod: 2022-06-17T20:44:32+08:00
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

## 分区概念

现在了解到安卓系统大概分三个分区：

- `system/priv-app`:系统预置分区
- `vendor/app`：厂商预置分区
- `data`:用户软件分区

平时我们靠adb安装的apk，都会装到`data`中，会覆盖掉另外两个分器的预置apk。

当卸载时，也只会卸载掉用户分区的apk，不会影响另外两个分区。当卸载掉用户分区的apk后，预置的同包名apk就会暴露出来，让人有种垃圾软件卸载不掉的感觉。

## 调试时的安装卸载方式

以下两种方式都只能卸载用户分区的apk，只有第三种实测可删除system和vendor中的apk。

### adb方式

安卓调试桥：

```shell
adb install [package name]
adb uninstall [package name]
```

### pm方式

PackageManager包管理器：

```shell
adb shell
pm install [package name]
pm uninstall [package name]

### shell方式

```shell
adb shell
cd xxxx
rm xxx.xx
```

### 获取root权限

```shell
adb root
```

### 重新挂载system分区

因为android系统的system分区在启动之后是只读分区，但在开发过程中需要对system分区进行修改，则需重新挂载成读写模式。

比如Android9.0启动了系统验证，因此在替换vendor目录内容时需要先禁止系统验证，并重新挂载system分区：

```shell
adb root 
adb remount
adb disable-verity
adb reboot
adb root
adb remount
```

### 文件传输

这里通常用来传输apk文件。

```shell
adb pull [目标设备文件] [本地路径]
adb push [本地文件] [目标设备路径]
```

### md5验证apk

由于替换system/vendor，我们不知道替换成功没有，这时就需要对比操作前后的apk的md5:

```shell
md5sum 文件
```

如果文件有改动，其md5必定与最之前的不同，说明替换成功。

### 抓log

由于现在很多东西都装在vendor，安装尚且困难，更别提挂端点调试了，也不会看到任何log，这时候就需要我们主动抓取log：

```shell

```