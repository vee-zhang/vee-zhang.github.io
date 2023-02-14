---
title: "Logcat"
subtitle: ""
date: 2022-07-06T22:55:20+08:00
lastmod: 2022-07-06T22:55:20+08:00
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
今天是我过得非常不爽的一天，因为今天说了『我不会』『我不懂』『我看不明白』，从哥2017年自学安卓到现在，从一点不会到基本不百度，我还没说过这些话。今天真是太刺激了，从上层转底层真的这么费劲吗，还是说年纪大了学不动了？都不是，是我一开始学系统底层还没找到要领，同时又事物缠身，分身无术。仔细想想还是先从常用工具入手，别他妈让我抓个log都这么费劲。
<!--more-->

## logcat是什么

就是Android-sdk提供的抓取log的工具，位于`Android/sdk/platform-tools/adb`.

## logcat启动方式

### adb方式

```
adb logcat
```

### shell方式

```
adb shell
logcat
```

## logcat帮助

```
adb logcat --help
```

## 缓冲区

| buffer  | description                    |
| ------- | ------------------------------ |
| radio   | 无线装置/电话短信              |
| events  | 二进制系统事件                 |
| main    | 主日志缓冲区，不包含系统和崩溃 |
| system  | 系统                           |
| crash   | 崩溃                           |
| all     | 所有                           |
| default | 默认，包含main、system、crash  |

## 命令格式

```
logcat [options] [filterspecs]
```

注意，这里的options可以有多个。

## 选项

| option          | description                                                                               |
| --------------- | ----------------------------------------------------------------------------------------- |
| -b              | 加载可供查看的日志缓冲区                                                                  |
| -g              | 输出日志缓冲区信息                                                                        |
| -c              | 清楚所选的缓冲区并退出，默认缓冲区集合为main、system、crash,清楚所有缓冲区则使用-b all -c |
| -e \<expr\>     | 抓取正则表达式匹配的行                                                                    |
| -m \<count\>    | 输出\<count\>行后退出，可与-e配合使用                                                     |
| -d              | 将缓冲区的log转存到屏幕后退出                                                             |
| -f \<filePath\> | 将日志写入文件，默认文件名为stdout，像不像std::cout                                       |
| -s              | 过滤，一会详细了解一下                                                                    |
| -v \<format\>   | 设置日志的输出格式，默认为threadtime                                                      |
| -D              | 输出日志时给各个缓冲区之间加个分割线来区分，推荐使用                                      |
| -t \<count\>    | 仅输出最新的行数，包含-d                                                                  |
| -t \<time\>     | 输出指定时间锚点以来的最新行，包含-d，如：adb logcat -t '01-26 20:52:41.820'              |
| -T              | 与-t一致，不包含-d                                                                        |
| -L              | 在最后一次重启之前转存log                                                                 |
| -B              | 以二进制文件输出日志                                                                      |
| -S              | 输出统计信息                                                                              |
| -G              | 设置日志缓冲区大小，单位是K或M                                                            |
| --pid=\<pid\>   | 仅输出指定pid的log                                                                        |

## 过滤

### 优先级与TAG

以下是log的优先级，与AS中对应：
| priority | description  |
| -------- | ------------ |
| V        | 详细         |
| D        | 调试         |
| I        | 信息         |
| W        | 警告         |
| E        | 错误         |
| F        | 严重错误     |
| S        | 静默，毛没有 |

TAG不用解释了，我懂得。

格式：

```shell
<priority>/<tag>
```

eg:

```
adb logcat ActivityManager:I MyApp:D *:S
```

显示tag为`ActivityManager`并且等级是`I`以上的日志；和tag为`MyApp`并且等级为`D`以上的日志；与此同时，通过`*:S`让其他日志静默。

>注意这里，实测用tag过滤后，并不能阻止其他日志的出现，所以后面必须加*:S才能单独显示tag标记后的日志。

## 格式控制

可以通过`-v`参数控制日志输出的格式。

| format     | description                     |
| ---------- | ------------------------------- |
| brief      | priority+tag+PID                |
| long       | 元数据字段，并使用空白行区分    |
| process    | PID                             |
| raw        | 不包含其他元数据的原始日志消息  |
| tag        | priority+tag                    |
| thread     | priority+PID+TID                |
| threadtime | 默认，time+priority+tag+PID+TID |
| time       | date+time+priority+tag+PID      |

## logcat导出

```
adb logcat -d >wzj.txt
```