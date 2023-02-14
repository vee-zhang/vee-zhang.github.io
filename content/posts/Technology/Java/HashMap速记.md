---
title: HashMap速记
date: 2021-01-12 17:49:10
tags: [java]
---

## 版本差异

| 版本   | 特点               |
| ------ | ------------------ |
| JDK1.7 | 数组+单向链        |
| JDK1.8 | 数组+单向链+红黑树 |

jdk1.7中，数组是HashMap的主体，而链表是为了解决哈希冲突而存在的。当链表过长，回严重影响性能，所以在1.8中，当链表超过8且数据总量超过64会转成红黑树：

- 链表超过8，且数据总量超过64会转成红黑树；
- 如果数组长度小于64，会优先扩容，以减少搜索时间。

## 默认加载因子

## Java中的二进制

### int的容量

我们一般使用的int是32位整型，占用4字节。在计算机中是以二进制来储存的，共有32位。

假设我们创建一个int i = 10，10的二进制是1010，一共只有4位，在计算机中会用0把前面补齐，满足32位存储，所以实际上计算机中储存的是：

```
00000000 00000000 00000000 00001010
```

### 源码，反码，补码

#### 源码

一个正数，按照**绝对值**大小转换成的二进制数；一个负数按照**绝对值**大小转换成的二进制数，然后**最高位改为1**，称为原码。

比如 00000000 00000000 00000000 00000101 是 5的 原码。

10000000 00000000 00000000 00000101 是 -5的 原码。

#### 反码

正数的反码与源码相同，负数的反码为对该数的原码**除符号位外**各位取反。

也就是原为1，得0；原为0，得1。（1变0; 0变1）。

5的反码：00000000 00000000 00000000 00000101

-5的反码：11111111 11111111 11111111 11111010

#### 补码

正数的补码与原码相同，负数的补码为对该数的原码除符号位外各位取反，然后在最后一位加1。

5的补码：00000000 00000000 00000000 00000101

-5的补码：11111111 11111111 11111111 11111011

### Java中对负数的处理

在二进制里，是用0和1来表示正负的，最高位为符号位，最高位为1代表负数，最高位为0代表正数。

**计算机以补码的形式存储数据。**

### 计算方式

- 正数：绝对值的二进制；
- 负数：绝对值转二进制，高位改为1，得到反码，再把反码+1得到补码。

## Java移位操作

平时看源码总遇到几个符号，很是阻碍我学习，所以趁这次回顾HashMap的源码的机会，顺便把移位也复习一下吧。

- << 左移位，将操作数向左移动指定的位数，并在低位补0；
- \>> 有符号右移位，将操作数向右移动指定的位数，若符号为正，则在高位插入0；若符号为负，则在高位插入1；
- \>>> 无符号右移位，无论正负，都在高位插入0。

我们以一个负数举例：int i = -10，分别求i左移位2、有符号右移位2和无符号右移位2后的值。

左移位：先把i=-10换算成二进制后的值为-1010，然后左移两位，低位用0补上，得到值为-101000.

有符号右移位：先把i=-10换算成二进制后的值为111111111111111111111111111 10110，然后开始右移，但由于空间不够，所以末尾的10就会被舍去，所以得到值为-10，因为符号为负，所以高位用1补上。

计算机的移位计算比乘除运算效率更高，所以Java源码中很常见移位操作。

## Java的&操作符

作用：如果相对应位都是1，则结果为1，否则为0。

实例：

```java
public static void main(String[] args) throws Exception {

    int i = 11;
    int j = 21;

    System.out.println(Integer.toBinaryString(i));
    System.out.println(Integer.toBinaryString(j));
    System.out.println(Integer.toBinaryString(i & j));
}
```

输出：

```
1011
10101
1
```

>大概了解一下就行，反正还是不会算。。。

## 源码解读

>一开始看的是JDK1.8的源码，好复杂，所以特意下载了JDK1.7的源码开始学习，以下都是基于1.7的。

### 类定义，支持克隆和序列化

```JAVA
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
```

这里要提以下，HashMap遵循`Map`、`Cloneable`、`Serializable`，所以**支持克隆，支持序列化**。

### 核心静态内部类——单向链

```java
static class Entry<K,V> implements Map.Entry<K,V> {

    //自己的key
    final K key;

    //自己的值
    V value;

    //下一节点
    Entry<K,V> next;

    //自己的哈希
    int hash;

    /**
    * 构造方法
    */
    Entry(int h, K k, V v, Entry<K,V> n) {
        ...
    }

    public final K getKey() {
        ...
    }

    public final V getValue() {
        ...
    }

    public final V setValue(V newValue) {
        ...
    }

    public final boolean equals(Object o) {
        ...
    }

    public final int hashCode() {
        ...
    }

    public final String toString() {
        ...
    }
}
```

源码中不重要的地方我都省略了。

从源码来看，`Entry<key,v>`类是一个单向链表结构。

>再一次看到这个`Entry`，不禁让我想起了前段时间研究AQS里面的`Node`。Node是一个双向链表形式的CLH队列，所以类名用Node表示，意思是队列中的一个**节点**；但是在HashMap中的Entry是一个单向链表，所以一个Entry只能充当入口来用。实在很佩服Java的作者，能把类名起的这么贴切！

### 构造方法

JDK1.7版本中构造方法共有4个，但是核心的只有两个，我们主要研究第一个：

```java
//常量，实际数值为1后面30个0,是个很大的数字，用于表示初始容量的极限
static final int MAXIMUM_CAPACITY = 1 << 30;
/**
 *核心构造方法一
 * @param initialCapacity 初始容量
 * @param loadFactor 加载因子
**/
public HashMap(int initialCapacity, float loadFactor){
    
    //初始容量不能小于0
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
    
    //初始容量降为极限
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    //加载因子不允许小于0，必须为数字
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + loadFactor);

    this.loadFactor = loadFactor;
    threshold = initialCapacity;

    //调用钩子方法，用来hook子类，本身没什么作用，需要子类重写才有用，但是我们一般也不重写。
    init();
}
```

构造方法最大的作用就是初始化**initialCapacity初始容量**和**loadFactor加载因子**，这两个字段有什么用，下面会介绍到。

### put方法

```java
//节点数组
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

public V put(K key, V value) {

    //初始化table数组
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    //key为null的处理方式
    if (key == null)
        return putForNullKey(value);

    //下面是key不为null的处理方式

    //计算key的哈希码
    int hash = hash(key);
    //找到hash码在数组中的位置，也就是查询定位
    int i = indexFor(hash, table.length);
    
    //遍历该位置的单向链，找到对应的key，赋值value
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;

    //如果一直找不到对应的key，就添加一个节点
    addEntry(hash, key, value, i);
    return null;
}

/**
 * key为null时的处理方式,只是没有计算哈希，而是直接从
 * table[0]位置开始遍历链表
 */
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}

/**
 * 根据hash找到数组中的对应位置
 */
static int indexFor(int h, int length) {
    return h & (length-1);
}
/**
 * 添加新节点
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        //如果当前size大于扩容因子，则扩容两倍
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

/**
 * 创建新节点，并插入队首
 */
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}

/**
 * 重新计算容量
 */
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

#### Put方法总结

##### 先看看流程

从Put方法可以看到，HashMap的核心是**table数组，数组的元素是Entry这个单向链**，这种结构被称之为**散列表**。

在存放数组时，使用`hash & (length-1)`的方式使数组均匀分布，提高查询效率。

>为何要均匀分布？因为查询时首先找到数组的对应下标，然后for循环对应的单向链。均匀分布使得链表长度尽可能小，减少了遍历次数，提高了效率。

如果查询没有获取结果，并且当前的size大于扩容因子，HashMap会先调用`resize()`方法进行扩容**翻倍**，最后创建一个新节点，插入**队首**，

**HashMap允许key、value为null**，当key为null时，会**默认存在table[0]**。

当key不为空时，在put时会依据key的哈希值找到数组中的对应位置，然后for循环遍历该位置的链表找到key，再进行储存。

##### 细说扩容

HashMap中table数组初始长度是16（默认状况下），而默认加载因子的值为0.75f，所以默认情况，当table的长度超过`16 * 0.75 = 12`的时候，再继续put就会翻倍扩容成32，然后再重新计算每个元素在数组中的位置，这是非常耗费性能的。

为了解决以上问题，我们在使用HashMap的时候，应该尽可能避免扩容，所以应该在初始容量上面做文章：

比如说，我们有1000个元素new HashMap(1000), 但是理论上来讲new HashMap(1024)更合适，不过上面annegu已经说过，即使是1000，hashmap也自动会将其设置为1024。 但是new HashMap(1024)还不是更合适的，因为0.75*1000 < 1000, 也就是说为了让0.75 * size > 1000, 我们必须这样new HashMap(2048)才最合适，既考虑了&的问题，也避免了resize的问题。 

[摘自](https://blog.csdn.net/sd_csdn_scy/article/details/55510453)https://blog.csdn.net/sd_csdn_scy/article/details/55510453

但是这样不可避免的出现了内存浪费。

##### 初始容量设置

- 必须是2的整数幂，可减少哈希冲突，使数据均匀分布，提高查询效率，并且可以避免空间浪费。
- 尽量在乘以0.75后大于预估元素数量。

### get 方法

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}

/**
 * key为null时的处理方式
 */
private V getForNullKey() {
    if (size == 0) {
        return null;
    }
    //还是直接去赵table[0]，然后遍历
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}

final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }

    //计算哈希值
    int hash = (key == null) ? 0 : hash(key);

    //查找哈希
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
            e != null;
            e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

get方法基本就没啥可说的了。

## 序列化

一开始看到类声明时，知道HashMap是支持序列化的，但是后来看到了：

```java
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```

table是`transient`的，是不允许序列化的。而且`Entry`是不支持序列化的，所以我就开始想HashMap是怎么进行序列化的呢？

### writeObject

```java
private void writeObject(java.io.ObjectOutputStream s)throws IOException
{
    // 写入容量、加载因子等信息
    s.defaultWriteObject();

    // 写入table数组当前的长度
    if (table==EMPTY_TABLE) {
        s.writeInt(roundUpToPowerOf2(threshold));
    } else {
        s.writeInt(table.length);
    }

    // 写入当前元素数量
    s.writeInt(size);

    // 遍历写入所有元素的KV
    if (size > 0) {
        for(Map.Entry<K,V> e : entrySet0()) {
            s.writeObject(e.getKey());
            s.writeObject(e.getValue());
        }
    }
}
```

### readObject

```java
private void readObject(java.io.ObjectInputStream s)throws IOException, ClassNotFoundException{
    
    s.defaultReadObject();
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
        throw new InvalidObjectException("Illegal load factor: " +loadFactor);
    }

    // set other fields that need values
    table = (Entry<K,V>[]) EMPTY_TABLE;

    // Read in number of buckets
    s.readInt(); // ignored.

    // Read number of mappings
    int mappings = s.readInt();
    if (mappings < 0)
        throw new InvalidObjectException("Illegal mappings count: " +
                                            mappings);

    // capacity chosen by number of mappings and desired load (if >= 0.25)
    int capacity = (int) Math.min(
                mappings * Math.min(1 / loadFactor, 4.0f),
                // we have limits...
                HashMap.MAXIMUM_CAPACITY);

    // allocate the bucket array;
    if (mappings > 0) {
        inflateTable(capacity);
    } else {
        threshold = capacity;
    }

    init();  // Give subclass a chance to do its thing.

    // Read the keys and values, and put the mappings in the HashMap
    for (int i = 0; i < mappings; i++) {
        K key = (K) s.readObject();
        V value = (V) s.readObject();
        putForCreate(key, value);
    }
}
```

### 序列化总结

HashMap的序列化和反序列化原理就是遍历读写table数组中的各个Entry元素的key和value，采用了`ObjectInputStream.readObject()`和`ObjectInputStream.writeObject()`方法。

## 知识点速记

1. 储存结构：
   | JDK版本 | 结构          | 特点                                    |
   | ------- | ------------- | --------------------------------------- |
   | 1.7     | 散列表        | 链表太长影响查询性能                    |
   | 1.8     | 散列表+红黑树 | 链表长度超过8并且数据总量超过64转红黑树 |
2. 核心字段：
    | 字段                      | 作用                                                                  |
    | ------------------------- | --------------------------------------------------------------------- |
    | `transient Entry[] table` | 主要存储，散列表                                                      |
    | `transient int size`      | entry的个数                                                           |
    | `int threshold`           | 阈值，等于table.length*loadFactory，table数组长度超过这个值就会扩容。 |
    | `loadFactory`             | 加载因子                                                              |
    | `modCount`                | 修改次数                                                              |
3. 初始容量：
    - 必须是2的整数幂，可减少哈希冲突，使数据均匀分布，提高查询效率，并且可以避免空间浪费。
    - 尽量在乘以0.75后大于预估元素数量。
4. put过程：
    先计算key的hash值，再根据`hash & (length-1)`找到key再table中的位置，遍历该位置上的链表找到对应的key；如果没有找到key，则`resize()`方法扩容，然后创建新的`entry`插入队首。
5. 关于扩容：
   - 条件：调用put方法时，size>threshold*loadFactory；
   - 方式：创建两倍长度的新数组，然后for嵌套while循环一个个的倒腾。
6. 关于序列化：
   自己实现了`writeObject`和`readObject`方法，通过反射调用；优先处理那几个核心字段，最后遍历entrySet分别读写key和value。
7. 关于EntrySet：
    EntrySet是HashMap的内部类，所以可以访问HashMap的成员，提供`nextEntry`方法，遍历的还是HashMap的散列表。

