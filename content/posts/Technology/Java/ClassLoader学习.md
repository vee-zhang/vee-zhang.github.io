---
title: ClassLoader学习.md
date: 2021-07-08 17:49:10
tags: [java]
description: 
---

## 用途

读取.class文件的字节码并加载成内存形式的类对象。

## 边解释边执行

JVM并不是一次性加载全部的类，而是需要用到才会去加载。比如加载了类，但是不会去加载类里面定义的对象类型的字段的类，因为暂时用不到。

## Java内置的ClassLoader

- `BootstrapClassLoader`
    负责加载 JVM 运行时核心类，这些类位于 JAVA_HOME/lib/rt.jar 文件中，我们常用内置库 java.xxx.* 都在里面，比如 java.util.*、java.io.*、java.nio.*、java.lang.* 等等。这个 ClassLoader 比较特殊，它是由 C 代码实现的，我们将它称之为「根加载器」。
- `ExtensionClassLoader`
    负责加载 JVM 扩展类，比如 swing 系列、内置的 js 引擎、xml 解析器 等等，这些库名通常以 javax 开头，它们的 jar 包位于 JAVA_HOME/lib/ext/*.jar 中，有很多 jar 包。
- `AppClassLoader` 
    这才是直接面向我们用户的加载器，它会加载 Classpath 环境变量里定义的路径中的 jar 包和目录。我们自己编写的代码以及使用的第三方 jar 包通常都是由它来加载的。
- `URLClassLoader`
    专门用来加载位于网络上静态文件服务器提供的 jar 包和 class文件。

## ClassLoader的选择

当我们定义一个ClassLoader，其内部都包含了一个ClassLoader，当遇到未知的类，JVM会选择调用者自己的ClassLoader去加载。

那么最初的调用者一定是持有`main()`方法的类，他持有的是`AppClassLoader`。

## 双亲委派

AppClassLoader只负责加载Classpath下面的类秒如果遇到没有加载过的系统类，他会将系统类库的加载工作交给`BootstrapClassLoader`和`ExtensionClassLoader`来做，这就叫**双亲委派**。

每个ClassLoader中都有一个`parent`属性指向父级，然后会优先让父级去加载，若父级加载失败，才轮到自己去加载：

AppClassLoader -> ExtensionClassLoader -> BootstrapClassLoader

> ExtensionClassLoader的parent的值为null，只要是parent=null的，说明父加载器就是BootstrapClassLoader。

## 使用ClassLoader

### Class.forName

```java
Class.forName("com.mysql.cj.jdbc.Driver");

Class<?> forName(String name, boolean initialize, ClassLoader cl)
```

这个方法默认使用“调用者”的ClassLoader来加载目标类，并且可以选择其他的ClassLoader。

### ClassLoader.loadClass

```java
ClassLoader.getSystemClassLoader().loadClass("[I");
```

## 使用

使用时一般都会结合反射。

```java
Class<?> clazz1 = App.class.getClassLoader().loadClass("Student");
Class<?> clazz2 = Student.class.getClassLoader().loadClass("Student");
Class<?> clazz3 = ClassLoader.getSystemClassLoader().loadClass("Student");
Class<?> clazz4 = Class.forName("Student");


Object obj1 = clazz1.getConstructor().newInstance();
clazz1.getMethods()[0].invoke(obj1, "obj1");

Object obj2 = clazz2.getConstructor().newInstance();
clazz1.getMethods()[0].invoke(obj2, "obj2");

Object obj3 = clazz3.getConstructor().newInstance();
clazz1.getMethods()[0].invoke(obj3, "obj3");

Object obj4 = clazz4.getConstructor().newInstance();
clazz1.getMethods()[0].invoke(obj4, "obj4");
```

## 自定义类加载器

ClassLoader 里面有三个重要的方法 loadClass()、findClass() 和 defineClass()。

- `ClassLoader` 不重写  用来加载目标类的入口。首先会查找当前自己和双亲是否已经加载目标类，没有的话就让双亲加载，弱失败，再调用`findClass()`。
- `findClass()` 需重写  去读取字节码，然后调用`defineClass()`。
- `defineClass()`   不重写  用来从字节码加载类对象。