---
title: Java趣味题
date: 2021-01-12 17:49:10
tags: [java]
description: 
---

## java是引用传递还是值传递？

>结论，Java就是值传递，只不过在传递引用类型的时候，会把对象的引用地址当作值来传递。

首先声明一个引用类型：

```java
private static class MyObject {

        private MyObject subObject;

        private String content;

        private int num;

        public MyObject(String content, int num) {
            this.content = content;
            this.num = num;
        }

        public MyObject(String content, int num, String subContent, int subnum) {
            this.content = content;
            this.num = num;
            this.subObject = new MyObject(subContent, subnum);
        }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder("当前的content是:" + content + "\t当前的num是" + num);
            if (subObject != null) {
                 sb.append("\n属性的content是:" + subObject.content + "\t属性的num是:" + subObject.num);
            }
            return sb.toString();
        }
    }
```

### 形参为一般类型，对实参无影响

```java
public static void main(String[] args) throws Exception {

    int num = 0;
    test(num);
    System.out.println("经test方法之后,num值为：\n" + num);
}
```

输出：

```
经test方法之后,num值为：
0
```

### 形参为引用类型，对实参无影响

```java
public static void main(String[] args) throws Exception {

    MyObject obj = new MyObject("我是第一层", 10, "我是第二层", 20);
        test(obj);
        System.out.println("经test方法之后,obj对象为：\n" + String.valueOf(obj));
    }

    /**
     * 我是方法
     * @param myobj
     */
    private static void test(MyObject myobj) {
        myobj = null;
    }
```

输出：

```java
经test方法之后,obj对象为：
当前的content是:我是第一层      当前的num是10 
属性的content是:我是第二层      属性的num是:10
```

可以看到在方法中给形参设置为null，但是依然可以输出。

### 改变形参的一般类型字段，会改变实参的字段

```java
private static void test(MyObject myobj) {
    myobj.num++;
}
```

输出：

```
经test方法之后,obj对象为：
当前的content是:我是第一层      当前的num是11
属性的content是:我是第二层      属性的num是:20
```

### 改变形参的引用字段，也会改变实参的字段

```java
private static void test(MyObject myobj) {
    myobj.content = null;
}
```

输出：

```
经test方法之后,obj对象为：
当前的content是:null    当前的num是10
属性的content是:我是第二层      属性的num是:20
```

### 改变形参引用字段内部的一般字段，实参会变

```java
private static void test(MyObject myobj) {
    myobj.subObject.num++;
}
```

输出：

```
经test方法之后,obj对象为：
当前的content是:我是第一层      当前的num是10
属性的content是:我是第二层      属性的num是:21
```

### 改变形参引用字段内部的引用字段，实参会变

```java
private static void test(MyObject myobj) {
    myobj.subObject.content=null;
}
```

输出：

```
当前的content是:我是第一层      当前的num是10
属性的content是:null    属性的num是:20
```

### 总结：

java是拷贝传递，如果形参是引用类型，则改变形参对实参无任何影响；而改变形参内部的属性，则会对实参产生影响。

## try里面return，是return先执行还是finally先执行？

```java
public static void main(String[] args) throws Exception {
    boolean result = test();
    System.out.print("收到结果为：" + result);
}

private static boolean test() {
    try {
        System.out.println("执行到了return");
        return true;
    } finally {
        System.out.println("执行到了finally");
    }
}
```

输出：

```
执行到了return
执行到了finally
收到结果为：true
```

从输出上看，应该是先return再执行finally，但是...

当我在return处打上断点，在finally内部输出那一行也打上断点，调试时惊喜的发现，先执行try内的输出，然后自动跳过了return语句直接在finally内部的断点停了，当我继续运行，发现它又回到了return语句，最后才离开方法。

### try和finally都return

```java
public static void main(String[] args) throws Exception {
    boolean result = test();
    System.out.println("收到结果为：" + result);
}

private static boolean test() {
    try {
        System.out.println("执行到了return");
        return true;
    } finally {
        System.out.println("执行到了finally");
        return false;
    }
}
```

当然到了现在，可以不相信输出了，要以断点顺序为准。

当try和finally中都有return时，Java再一次给了我惊喜：

首先执行了try中的return，然后又执行了finally中的return，最后才返回结果，并且是以finally中的return为准！

结论：

- finally中没有return，先finally再try的return；
- finally中有return，先try的return，再finally中的return，结构以后执行的return为准。

所以**Java无论如何都是先finally再return**！