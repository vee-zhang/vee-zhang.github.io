---
title: "C语言的Char"
subtitle: ""
date: 2022-12-15T17:29:25+08:00
lastmod: 2022-12-15T17:29:25+08:00
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

## 字符

### 专用打印方式

```
char a = '1';
putchar(a);
printf("%c",a);
```

`putchar`函数单次只能输出一个char,相对来说`printf`函数要强大的多。

### 字符与整数

**计算机在存储字符时并不是真的要存储字符的实体，而是存储该字符在字符集中的编号**。对于`char`类型来说，世纪存储的就是字符的**ASCII**码。

无论在哪个字符集中，字符编号都是一个**整数**；从这个角度去考虑，字符类型本质上与整数一致。

**我们可以给字符类型赋值一个整数，或者以整数的形式输出字符类型。反过来，也可以给整数类型赋值一个字符，或者以字符的形式输出整数类型。**

```
#include <stdio.h>
int main()
{
    char a = 'E';
    char b = 70;
    int c = 71;
    int d = 'H';

    printf("a: %c, %d\n", a, a);
    printf("b: %c, %d\n", b, b);
    printf("c: %c, %d\n", c, c);
    printf("d: %c, %d\n", d, d);

    return 0;
}

输出：

a: E, 69
b: F, 70
c: G, 71
d: H, 72
```

### 中文字符

由于char使用电脑系统的默认编码，一般windows上都是ASCII，只能识别到英文字母，所以是不支持汉字单字符的。所以要存储汉字字符，需要使用**宽字符wchar_t**。

宽字符wchar_t使用`UTF-16`或`UTF-32`编码。

### 字符的编码

char

## 字符串

C语言中压根就没有字符串，只能使用**数组**和**指针**来渐渐的储存字符串。

```
//数组方式
char str1[] = "http://c.biancheng.net"; //这里帮原作打个广告
//指针方式
char *str2 = "C语言中文网";
```