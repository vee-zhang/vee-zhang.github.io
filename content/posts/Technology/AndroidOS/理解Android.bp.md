---
title: "理解Android"
subtitle: ""
date: 2022-07-15T13:32:52+08:00
lastmod: 2022-07-15T13:32:52+08:00
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

## 鸣谢

本文是边学官文和博客[静默安装9339的文章](https://blog.csdn.net/fanx9339/article/details/106896653)边做的整理，感谢原作！

## 作用

## c/c+语言对编译器的选择

| language | build-tool  |
| -------- | ----------- |
| c        | gcc,clang   |
| c++      | g++,clang++ |

这里推荐选择`clang`系列，clang系列的效率更高，编译产物也更小。

## make的演变过程

上面这些编译器都只能编译一个文件，所以当我们跑项目，有多个文件的时候，就需要做个批处理。

最早出现了`make`工具，通过编写`makefile`作为文件配置，实现了批处理。

但项目越来越大，文件越来越多，makefile编写也会越来越复杂，所以聪明的老外又弄出一个`cmake`工具，pregrammer们可以用简化后的`cmakefile`，通过这个工具转化为makefile，并且`cmakefile`中可以加入一些判断和循环等逻辑，帮我们省了不少工作量，最后**cmake还是跨平台的**。

然后google的大佬在编译越来越庞大的源码时，慢到不能忍，他就怪到了cmake的头上，所以做出一个`Ninja`来替换cmake，用`.ninja`文件替换`cmakefile`。**所以Ninja与cmake是平级的**。

最后，Ninja和cmake还是基于目录和文件的思维来组织项目（很有c的传统），google又弄出了个`Soong`来**以「模块module」思维组织项目**，同时带来了`Android.bp`文件，其内容描述的是模块结构，然后编译成`.ninja`文件，最后再由Ninja来实际编译。

>说白了就是层层封装，由于到Android10.0的源码中还存在Android.mk,所以带给我们的就是需要把make、ninja和soong都学会才能解决复杂问题，然后soong还是go语言写的，所以我还得学个go，真google！

## 文件格式

我现在的理解，文件名必须是`Android.bp`。

```
cc_binary {
    name: "gzip",
    srcs: ["src/test/minigzip.c"],
    shared_libs: ["libz"],
    stl: "none",
}
```

>内容就是一个一个的模块，也没啥文件声明、版本号什么的，这点给好评。

### 语法

```
moduleType {
  name:value,
  srcs:[
    "src/test/minigzip.c",
    "src/test/minigzip.c"
  ]
}
```

每个模块由`模块类型`和`模块的属性`两部分组成，大括号里面的就是属性。

- `moduleType`是固定的那几个，不能乱写，可在源码中`android/build/soong/androidmk/cmd/androidmk/android.go`中查看，也可以参考[google慢成屎的网站](https://ci.android.com/builds/submitted/8833803/linux/latest/view/soong_build.html)
- `name`是每个模块的**必填项**，并且**文件内唯一**。
- `srcs`用于指定构建模块的源文件。[todo] 模块引用语法

### 默认模块

`cc_defaults`类型的模块可用于在多个模块中**复用属性**，如：

```
cc_defaults {
    name: "gzip_defaults",
    shared_libs: ["libz"],
    stl: "none",
}

cc_binary {
    name: "gzip",
    defaults: ["gzip_defaults"],
    srcs: ["src/test/minigzip.c"],
}
```

### 预构建模块

- [] 待完善

### 命名空间

- [] 待完善

## 逻辑编程

### 类型

这里的变量和属性是`强类型`的，以初始化赋值时的类型来确定。

- 布尔
- 整型
- 字符串
- 字符串数组
- 映射表（类似HashMap）

### 通配符

**接受文件列表的属性**（如srcs）可以采用glob写法。

这种写法可以包含通配符`*`，如`*.java`。

- `*`匹配任意文件名，如`*.java`；
- `**`匹配0～n个路径，如`java/**/*.java`同时匹配 java/Main.java 和 java/com/android/Main.java。

### 变量

支持顶级变量形式：

```
gzip_srcs = ["src/test/minigzip.c"],
cc_binary {
    name: "gzip",
    srcs: gzip_srcs,
    shared_libs: ["libz"],
    stl: "none",
}
```

变量只在文件内生效，且**无法改变**。可以通过`+=变量`的形式为其他变量赋值。

### 注释

- 单行`//`
- 多行`/* */`

### 运算符

>可以使用 + 运算符附加字符串、字符串列表和映射。可以使用 + 运算符对整数求和。附加映射会生成两个映射中键的并集，并附加在两个映射中都存在的所有键的值。

简言之就是只有一个`+`号能用。

### 条件

Soong不支持条件判断，需要额外写个`go`文件实现。然后条件语句会转换为**映射属性**，再把映射属性赋值给模块的属性。

比如,要区别不同的cpu架构：

```
cc_library {
    ...
    srcs: ["generic.cpp"],
    arch: {
        arm: {
            srcs: ["arm.cpp"],
        },
        x86: {
            srcs: ["x86.cpp"],
        },
    },
}
```

还不会Go,先掠过。。。

- [] 学习Go,再回来补充这里。