---
title: Travis持续发布Hugo博客
subtitle: ""
date: 2021-07-22T17:01:25+08:00
lastmod: 2021-07-22T17:01:25+08:00
draft: false
authors: []
description: ""

tags: [Travis,Hugo]
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

以前使用`Hexo`的时候了解到了`Travis`这个好东东，应该是我自jenkins之后接触过的第二个CI了，挺有意思。

之前的做法，或者说一般网上的做法，都是只弄一个仓库，然后源码和生成物在不同的分支，包括Travis官文上也是这样写的。这就存在两个问题：
  1. Travis只有发布到开源仓库才免费，而发布到私有仓库每月好像要100多美元；
  2. 如果发布到开源仓库的不同分支，那我私密性的博客不就源码可见了嘛。

所以我这里采用了**两个仓库**：

1. 源码仓库，私有，只用来存放源码；
2. 静态文件仓库，开源，只用来存放Hugo的生成物。

.travis.yml

```yml
language: go

go:
  # 指定Golang 1.8
  - "1.8" 

env:
  global:
    # 定义一个全局变量
    - GH_REF: github.com/vee-zhang/vee-zhang.github.io

# 分支白名单限制: 只有master分支的提交才会触发构建
branches:
  only:
    # 指定master分支参与构建
    - master

install:
  # 安装hugo
  - wget -q -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v0.86.0/hugo_extended_0.86.0_Linux-64bit.deb
  - sudo dpkg -i hugo.deb

script:
  # 运行hugo命令
  - hugo

  # git直接推public
  - cd public
  - git init
  - git config user.name "vee-zhang"
  - git config user.password "$githubtoken"
  - git add -A
  - git commit -m "Update Blog By TravisCI With Build $TRAVIS_BUILD_NUMBER"
  - git push --force --quiet "https://$githubtoken@${GH_REF}" master:master
```

我通过各种方法来安装Hugo，包括`go get`直接安装，clone源码然后`go install`编译安装，都报错失败了。最后从网上查到竟然可以使用wget下载成品！！！

但是这样以来确定也很明显，就是Hugo不能自动更新了。留待以后解决吧。

## 添加Travis构建结果图标

<img src="https://www.travis-ci.com/vee-zhang/VeeBlogSourceCode.svg?token=zr2k46J9vsyyAMcKzMGx&amp;branch=master&amp;status=passed" alt="build:passed">

```html
<img src="https://www.travis-ci.com/vee-zhang/VeeBlogSourceCode.svg?token=zr2k46J9vsyyAMcKzMGx&amp;branch=master&amp;status=passed" alt="build:passed">
```

鸣谢：

[qq_43103581](https://blog.csdn.net/qq_43103581/article/details/103203605)