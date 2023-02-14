---
title: "Java中的范型"
subtitle: ""
date: 2022-03-19T15:33:02+08:00
lastmod: 2022-03-19T15:33:02+08:00
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

## 伪范型和范型擦除

**在编译期间，所有的范型信息都会被擦除掉。**

Java中的范型是在编译器这个层次来实现的，在生成Java字节码中是不包含范型中的类型信息的，使用范型的时候加上的类型参数，会在编译期间去掉，这个过程就叫范型擦除。

验证：声明两个不同范型的List，然后比较两个list的类型。

```java
ArrayList<String> list1 = new ArrayList();
ArrayList<Integer> list2 = new ArrayList();

System.out.print(list1.getClass() == list2.getClass());

// 输出 true
```

## 范型通配符

- `<? extends T>` 子类限定通配符，
- `<? super T>` 超类限定通配符，
- `<?>`  无限定通配符，

## 范型的限制

1. 范型不能用来实例化变量；
2. 静态域或方法不能使用自己类型的范型（因为只有在创建对象后才知道范型的具体类型）；
3. 范型不支持使用结构体，可以使用引用的包装类型替代，如Double,Integer,Long等；
4. 范型不允许使用instanceof关键字判断类型；
5. 类型擦除；
6. 范型类不能继承/扩展（使用extends关键字）Expection、Throwable；
7. 不能try-catch捕获范型对象；