---
title: "vscode+gradle+kotlin"
subtitle: ""
date: 2022-07-07T23:47:59+08:00
lastmod: 2022-07-07T23:47:59+08:00
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
最近事多，整天瞎忙，都是沟通、开会、填表格和填坑，今天想休息休息，打算有空再写写后端服务，把自己的网盘和图床先弄好。思来想去，自己一个人的项目，时间少，所以尽量还是要方便维护，那么就不要用JAVA了，罗里吧嗦的，可惜Dart这边没什么好的框架，要不然一个语言全站了，最终还是选择了kotlin，但是又不想用idea，于是我瞄向了vscode...
<!--more-->

## IDE和技术选型

由于idea太贵了，今年又涨价了，并且破解经常失效，总是突然把我搞的焦头烂额的，所以很反感这类收费的IDE。

后来转回eclipse，发现这货慢、丑、笨，于是又开始尝试宇宙第一IDE的小老弟——vscode。

一开始，尝试用vscode写了几个java微服务，真叫香啊，仿佛就是天生为SpringCloud而生的。但是Spring本身生态混乱，庞大，笨拙，比如注册中心我采用阿里的nacos，鉴权中心采用keycloak，这俩货本身是脱离SpringCloud而独立存在的，再加上熔断、网关等等，我的项目就是家里几个人用，我何必这么卷自己呢？况且我的小树莓派也不一定吃的消。

后来陆续又尝试了Dart的两个后端框架，官方维护的跟狗屎一样，又尝试了红帽的Quarkus，也是太庞大复杂了。

最后又看了一下Kotlin这边倒是出现了一个新框架叫`ktor`，设计的挺清奇的，用起来也还不错，生态基本也算健全。

那么现在唯一的问题就是IDE的选择了。在网上搜vscode开发kotlin，基本都是基于codeRunner，这个东西实在太幼稚了...于是乎，一年过去了我的项目还没落地，艹！

时间来到最近，我看了几篇在vscode上搭建c++环境的文章，根本不需要什么codeRunner，今天闲下来突然灵机一动，我照猫画虎，终于搞定了！✌🏻
## 插件选择

需要[这个](https://marketplace.visualstudio.com/items?itemName=fwcd.kotlin)和[这个](https://marketplace.visualstudio.com/items?itemName=richardwillis.vscode-gradle-extension-pack)

目前最好用的kotlin-languageServer，还带代码补全（聊胜于无）和lint还有debug。

另一个是巨硬官方的gradle插件，稳定性可靠。

## gradle环境变量

在[这里](https://services.gradle.org/distributions/)下载gradle，尽量选择正式的高版本的，解压，然后设置环境变量：

```
export GRADLE_HOME=/Users/vee/Library/gradle-7.4.2
export PATH=$PATH:$GRADLE_HOME/bin
```

>windows怎么设置自己去查，我不会用windows。

然后终端`source .profile`

最后终端执行命令`gradle`看看成功没。

## gradle创建kotlin项目

```
gradle init --type kotlin-application
```

跟着提示一直走。

## vscode配置

`cd`到项目根目录，然后直接`code .`，就可以在vscode中打开项目了。

然后vscode的菜单栏->终端->配置默认生成任务-》gradle:app:build，一顿操作猛如狗，就会发现项目根目录多了一个`.vscode/tasks.json`,如果是多moudle工程，可以在这个文件的tasks数组里面照猫画虎再添加个task就行了。

```
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "gradle",
			"id": "/Users/vee/Downloads/testapp:buildapp",
			"script": "app:build",
			"description": "Assembles and tests this project.",
			"group": {
				"kind": "build",
				"isDefault": true
			},
			"project": "app",
			"buildFile": "/Users/vee/Downloads/test/app/build.gradle.kts",
			"rootProject": "test",
			"projectFolder": "/Users/vee/Downloads/test",
			"workspaceFolder": "/Users/vee/Downloads/test",
			"args": "",
			"javaDebug": false,
			"problemMatcher": [
				"$gradle"
			],
			"label": "gradle: app:build"
		}
	]
}
```

接下来在相同目录下创建一个`launch.json`文件，文件内容如下：

```
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "kotlin",
            "request": "launch",
            "name": "Kotlin Launch",
            "projectRoot": "${workspaceFolder}/app",
            "mainClass": "MyKotlinTest.AppKt",
            "preLaunchTask": "gradle: app:build"
        }
    ]
}
```

首先添加`preLaunchTask`字段，值就是刚才task的label，要原封不动的复制过来。

然后修改`projectRoot`字段，他默认是"${workspaceFolder}"，这是vscode中的环境变量，表示项目的根目录。但是gradle的文件结构是多了一层app目录的，所以需要在后面添加`app`路径。

最后修改`mainClass`字段，这个字段表示程序入口类，值是入口类的全名称（包名+文件名）。由于是kotlin类型的文件，后缀名都是`.kt`，所以类名要加上`Kt`后缀，这是kotlin插件要求的，真tm思路清奇。

最后放心直接按F5，当然我的touch bar上直接显示了play按钮，点他一下，就可以看到程序自动build，并且运行了✌🏻

## 后记

9点回来，弄好环境，去洗了个香香，然后洗袜子，洗内裤，拖地，回来写blog，一看表又12点半了，睡吧，明天继续去编译系统了。