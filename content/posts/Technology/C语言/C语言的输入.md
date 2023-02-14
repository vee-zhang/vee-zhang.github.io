---
title: "C语言的输入"
subtitle: ""
date: 2022-12-16T14:45:39+08:00
lastmod: 2022-12-16T14:45:39+08:00
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

## scanf()函数

`scanf()`函数用于格式化扫描，与`print()`函数正好相对。

```c
#include <stdio.h>
int main()
{
    int a = 0, b = 0, c = 0, d = 0;
    scanf("%d", &a);  //输入整数并赋值给变量a
    scanf("%d", &b);  //输入整数并赋值给变量b
    printf("a+b=%d\n", a+b);  //计算a+b的值并输出
    scanf("%d %d", &c, &d);  //输入两个整数并分别赋值给c、d
    printf("c*d=%d\n", c*d);  //计算c*d的值并输出
    return 0;
}

运行后在控制台中输入：

12回车
60回车
输出：a+b=72

再输入：
10 23回车
输出：c*d=230
```


