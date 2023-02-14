---
title: "类图"
subtitle: ""
date: 2022-06-08T09:20:05+08:00
lastmod: 2022-06-08T09:20:05+08:00
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
## yuml类图基本格式

```
// {type:class}
// {direction:topDown}
// {generate:true}

[note: You can stick notes on diagrams too!{bg:cornsilk}]

[Customer]<>1-orders 0..*>[Order]
[Order]++*-*>[LineItem]
[Order]-1>[DeliveryMethod]
[Order]*-*>[Product|EAN_Code|promo_price()]
[Category]-.->[Product]
[DeliveryMethod]^[National]
[DeliveryMethod]^[International]
```

上面是个列子。

| Item                    | Example                                       |
| ----------------------- | --------------------------------------------- |
| Class                   | [Customer]                                    |
| Directional(单向关联)   | [Customer]->[Order]                           |
| Bidirectional(双向关联) | [Customer]<->[Order]                          |
| Aggregation(聚合)       | [Customer]+-[Order] or [Customer]<>-[Order]   |
| Composition(组合)       | [Customer]++-[Order]                          |
| Inheritance(泛化)       | [Customer]^[Cool Customer]                    |
| Dependencies(依赖)      | [Customer]uses-.->[PaymentStrategy]           |
| Cardinality             | [Customer]<1-1..2>[Address]                   |
| Labels                  | [Person]customer-billingAddress[Address]      |
| Notes                   | [Address]-[note: Value Object]                |
| Full Class              | `[Customer                                    |
| Color splash            | [Customer{bg:orange}]<>1->\*[Order{bg:green}] |

## 访问修饰符

| 修饰符 | 描述      |
| ------ | --------- |
| +      | public    |
| -      | private   |
| #      | protected |
| 无     | package   |

## 关系

### 依赖关系

表示一种临时性的，借助外部事物的【使用】关系，比如我们要打电话，此时需要一台手机，那么就是person依赖mobilePhone，但是人的身体里并没有手机，手机是外部的。一般在代码中的体现就是，某个类通过调用其他类的某些方法完成任务，叫做”某个类“依赖”其他类”。

```
// {type:class}
[Customer]uses-.->[PaymentStrategy]
```

### 泛化关系

泛化关系式对象之间耦合度最大的关系，表示一种扩展（继承）关系，是符合`is-a`的关系。

表示为空心箭头：

```
// {type:class}
[Student]^[Person]
```

### 实现关系

接口与实现类的关系。

表示为虚线+空心箭头。

```
// {type:class}
[Student]^[Person]
```

### 关联关系

是对象之间的一种引用关系，表示一类对象与另一类对象之间的联系。JAVA中一般指类内部包含的属性，而这个属性一般是其他类型。

#### 单向&双向关联

在代码中通常将一个类的对象作为另一个类的成员变量来实现关联关系。

```
[Customer]<->[Order]
```

表示为实线+实心箭头指向被关联的类。

#### 聚合

聚合（Aggregation）关系是关联关系的一种，是强关联关系，是整体和部分之间的关系，是 has-a 的关系。

聚合关系也是通过成员对象来实现的，其中成员对象是整体对象的一部分，但是**成员对象可以脱离整体对象而独立存在**。例如，学校与老师的关系，学校包含老师，但如果学校停办了，老师依然存在。

表示为实线+空心菱形（指向整体）。

```
[Teacher]+-[University]
```

#### 组合

表示类之间的“整体”与“部分”的关系。是`contains-a`关系。

在组合关系中，整体对象可以控制部分对象的生命周期，一旦整体对象不对在，部分对象也将不存在，**部分对象不能脱离整体对象而存在**。

表示为实线+实心菱形指向整体。