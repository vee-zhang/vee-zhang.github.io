---
title: "Hugo的图片处理"
subtitle: ""
date: 2022-01-17T17:13:10+08:00
lastmod: 2022-01-17T17:13:10+08:00
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

用了Hugo这一段时间，一直搞不明白图片怎么存放比较好，也一直弄不清楚**Leaf Bundle**和**Branch Bundle**有什么区别，今天有点时间，终于搞清楚了，并且图片也成功在Github-pages上显示出来了，特此记录一下吧！
<!--more-->

### Hugo的图片存放方式

Hugo的图片一般有两种存放方式：

- 放在`static`目录中成为静态资源，在引用时可以以当前目录地址直接引用。比如我在`static`中创建了一个`images`目录，把所有的图片都放在里面，然后可以这样引用：
    ```
    ![图片](/images/xxx.png)
    ```
    这种方式的问题很大：

    1. 图片没有分门别类，全都放在一起，很难维护；
    2. 由于用的是编译后的相对位置，所以在编写文章时无法实时预览，很容易导致语法错误而难以排查。
- 利用`Leaf Bundle`存放。
    好处是图片与文章在同一目录下，结构清晰利于维护，并且在编辑时所见即所得。

### Page Bundle

Hugo的PageBundle只有两种：

- 叶子节点，其所在目录中只能有一个`index.md`文件，用来写文章内容，其余文件都被认为是静态资源；
- 枝条节点，其所在目录中只能有一个`_index.md`文件，用以告诉Hugo这个文件夹是个目录，所在文件夹内可包函其他叶子节点或常规文章。

### 利用Leaf Bundle维护图片

比如我的目录结构：

```
├─ content
│  ├─ about.md
│  └─ posts
│     ├─ Technology
│     │  ├─ Android
│     │  │  ├─ Android应用启动流程
│     │  │  │  ├─ index.md
│     │  │  │  └─ 启动流程.png
│     │  │  ├─ _index.md
│     │  ├─ _index.md
```

其中`Technology`目录中有_index.md，所以是枝条节点，里面又有枝条节点`Android`，再其中，`Android应用启动流程`只有一个index.md和一张图，是叶子节点。我们就在这个index.md中写内容，直接引用图片`启动流程.png`的相对地址就可以了，代码如下：

```
## 流程

![](启动流程.png)
```

这样就可以使图片所见即所得的编写博客文章了。