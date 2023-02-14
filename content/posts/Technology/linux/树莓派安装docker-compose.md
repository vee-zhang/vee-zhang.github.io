---
title: "树莓派安装docker-compose"
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

过年这几天鼓捣一下树莓派，结果各种麻烦，尤其是安装docker-compose，明明docker安装很顺畅，为啥这货就这么麻烦呢？
<!--more-->

结果反复尝试，终于找出了问题的关键。docker-compose的官网是通过从github下载安装的，具体指令是：

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

就是这个指令坑了我很久。

后来查到了用`pip`来安装的方法，单好不容易把pip安装上之后，还是不行。

最后看到了[这篇文章](https://www.cnblogs.com/xie37/p/15778519.html)，感谢原作者！

我才发现，原来是我的url有问题，首先树莓派上的ubuntu-server应该安装`aarch64`类型的docker-compose，正确的指令如下：

```
sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-aarch64" -o /usr/local/bin/docker-compose
```
看看，对比下来发现官网的url里面的`${uname}`只不过是个占位符而已。

最后更换了url地址之后完美解决。