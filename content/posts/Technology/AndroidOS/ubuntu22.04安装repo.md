---
title: "Ubuntu22.04安装配置repo"
subtitle: ""
date: 2022-06-29T20:01:30+08:00
lastmod: 2022-06-29T20:01:30+08:00
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

## 安装Repo

### apt安装（不推荐）

```
sudo apt-get update
sudo apt-get install repo
```

实测apt安装非常方便，而且能自动配置好环境变量。但是ubuntu22.04LTS源上的版本是比较旧，对我这种新潮党很不友好。

### 下载安装

在用户根目录下创建一个bin目录：

```
mkdir ~/bin
```

不需要修改，直接执行以下命令：

```
export REPO=$(mktemp /tmp/repo.XXXXXXXXX)
curl -o ${REPO} https://storage.googleapis.com/git-repo-downloads/repo
gpg --recv-key 8BB9AD793E8E6153AF0F9A4416530D5E920F5C65
curl -s https://storage.googleapis.com/git-repo-downloads/repo.asc | gpg --verify - ${REPO} && install -m 755 ${REPO} ~/bin/repo
```

这些命令会设置一个**临时文件**，将 Repo 下载到该文件中，并验证提供的密钥是否与所需的密钥匹配。如果这些步骤成功完成，就会继续进行安装。

>mktemp命令可查看https://www.runoob.com/linux/linux-comm-mktemp.html

这一步执行完毕之后，在`bin`目录下就会出现一个`repo`文件，它是个python可执行文件。

我们用任何编辑器打开它，比如我喜欢vscode：

```
cd ~/bin
code repo
```

在文件顶端第一行有格式声明，我下载的repo用的是python2,但是我的ubuntu只安装了python3,所以需要把2改成3：

```
#!/usr/bin/env python
改为：
#!/usr/bin/env python3
```

之后，由于repo在初始化时默认访问google地址，速度非常慢，所以需要改成清华的镜像，在repo文件内把`https://gerrit.googlesource.com/git-repo`替换成`https://mirrors.tuna.tsinghua.edu.cn/git/git-repo`，并保存。

```
REPO_URL = os.environ.get('REPO_URL', None)
if not REPO_URL:
  # REPO_URL = 'https://gerrit.googlesource.com/git-repo'
  REPO_URL = 'https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
REPO_REV = os.environ.get('REPO_REV')
if not REPO_REV:
  REPO_REV = 'stable'
# URL to file bug reports for repo tool issues.
BUG_URL = 'https://bugs.chromium.org/p/gerrit/issues/entry?template=Repo+tool+issue'
```

>Python是一种强格式语言，所以替换的时候一定要注意缩进的位数与之前一致。

最后查看一下repo版本号，验证是否安装成功：

```
repo version
```

## 下载安卓源码

### 初始化Repo

创建一个空目录来存放您的工作文件。为其指定一个您喜欢的任意名称：

```
mkdir WORKING_DIRECTORY
cd WORKING_DIRECTORY
```

git需要配置全局名称和邮箱：

```
git config --global user.name Your Name
git config --global user.email you@example.com
```

运行`repo init`更新repo,并为repo指定一个网址：

```
repo init -u https://android.googlesource.com/platform/manifest
```

据说从google仓库下载会非常慢，而且很容易失败，幸好公司这边已经有了😀：

```
repo init -u "git://10.53.131.102/platform/manifest" -b android-9.0.0_r46   
```

最后同步Android源码：

```
repo sync
```

更快的方式，加上参数:

- -c 表示从当前分支同步
- -j8 表示启动8个线程并发同步

```
repo sync -c -j8
```

执行之后，发现好慢，抽烟去了。