---
title: "LruCache"
subtitle: "Android的内存缓存"
date: 2021-09-15T14:18:13+08:00
lastmod: 2021-09-15T14:18:13+08:00
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
LRU这个算法就是把最近一次使用时间离现在时间最远的数据删除掉，而实现LruCache将会频繁的执行插入、删除等操作，我们就会想到使用LinkedList，但是我们又要基于Key-Value来保存数据，这个时候我们就会想起HashMap，但是HashMap不能像linkedList那样保留数据的插入顺序，如果要使用HashMap的话可以使用它的一个子类LinkedHashMap。
<!--more-->

### LinkedHashMap

可理解为有序的HashMap。

#### 构造方法

```java
// 队首
private transient LinkedHashMapEntry<K,V> header;

// 排序模式
private final boolean accessOrder;


public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {
  ...
}
```

这货是继承自`HashMap`的，那么他就有HashMap的所有特性，包括线程不安全和翻倍扩容。

然后来看一下他内部的entry：

#### 入口Entry

```java
/**
* 继承自HashMap的Entry
*/
private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
    
    // 内部实现了自己的before和after，成了一个双向链表
    LinkedHashMapEntry<K,V> before, after;

    ...
}
```

可以看到，他内部的`Entry`同时具备双向链和单向链(继承了HashMap的entry内部的next字段)结构。
  - next是用于维护HashMap指定table位置上连接的Entry的顺序的；
  - before、After是用于维护Entry插入的先后顺序的。

#### 核心构造方法

```java
// 排序模式 true-访问排序  false-插入排序
private final boolean accessOrder;

public LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder) {
    super(initialCapacity, loadFactor);

    this.accessOrder = accessOrder;
}
```

与HashMap的主要区别就是多了`accessOrder`属性，它的作用是选择排序模式：
  - true  访问排序
  - false 插入排序

这里有个坑要注意的是，调用了父类的构造函数：

```java
public HashMap(int initialCapacity, float loadFactor) {
 
    if (initialCapacity > MAXIMUM_CAPACITY) {
        initialCapacity = MAXIMUM_CAPACITY;
    } else if (initialCapacity < DEFAULT_INITIAL_CAPACITY) {
        initialCapacity = DEFAULT_INITIAL_CAPACITY;
    }
    threshold = initialCapacity;

    // 这个init方法好坑啊
    init();
}

void init() {}
```

其中调用了`init`方法，而HashMap中并没有实现这个方法，说明是专门用来给子类初始化的，于是我去看LinkedHashMap中的重写实现：

```java
@Override
void init() {
  header = new LinkedHashMapEntry<>(-1, null, null, null);
  header.before = header.after = header;
}
```

初始化了队首的entry，并且它的before和after都是它自身。

#### put方法

LinkedHashMap并没有重写`put`方法，所以他是**直接沿用了HashMap的put方法**。

来回顾一下HashMap的put方法：

```java
public V put(K key, V value) {
  if (table == EMPTY_TABLE) {
      inflateTable(threshold);
  }
  if (key == null)
      return putForNullKey(value);
  
  // 计算Key的Hash值
  int hash = sun.misc.Hashing.singleWordWangJenkinsHash(key);

  //找到key的下标位置
  int i = indexFor(hash, table.length);

  //遍历单向链，插入到对应的hash位置
  for (HashMapEntry<K,V> e = table[i]; e != null; e = e.next) {
      Object k;
      if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
          V oldValue = e.value;
          e.value = value;
          e.recordAccess(this);
          return oldValue;
      }
  }

  //没有对应则插到末尾
  modCount++;
  addEntry(hash, key, value, i);
  return null;
}
```

这里有一点要注意的是，LinkedHashMap大量运用了**多态**，所以在添加的时候调用的`addEntry`方法，实际上是LinkedHashMap已经重写过的：

```java
void addEntry(int hash, K key, V value, int bucketIndex) {

    LinkedHashMapEntry<K,V> eldest = header.after;

    // 如果after不是自己
    if (eldest != header) {
        boolean removeEldest;
        size++;
        try {
            removeEldest = removeEldestEntry(eldest);
        } finally {
            size--;
        }
        if (removeEldest) {
            removeEntryForKey(eldest.key);
        }
    }

    super.addEntry(hash, key, value, bucketIndex);
}
```

然后又调用父类的`addEntry`方法，而在其中再次调用自己的多态方法`createEntry`：

```java
void createEntry(int hash, K key, V value, int bucketIndex) {

    HashMapEntry<K,V> old = table[bucketIndex];
    LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
    table[bucketIndex] = e;
    // 同时指定了after和before
    e.addBefore(header);
    size++;
}

// LRU算法，把当前entry插入到header队首之前，成为新的队首
private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
    // 插入到header之前
    after  = existingEntry;
    before = existingEntry.before;
    before.after = this;
    after.before = this;
}
```

创建了一个entry，插入到header与old之间，再把size+1，这样就形成了**插入顺序**。

#### get方法

```java
public V get(Object key) {

  // 先通过key拿到entry
  LinkedHashMapEntry<K,V> e = (LinkedHashMapEntry<K,V>)getEntry(key);
  
  // 判空
  if (e == null) return null;
  
  e.recordAccess(this);
  return e.value;
}

// 记录下访问操作
void recordAccess(HashMap<K,V> m) {
  LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
  // 如果是访问排序，就把查到的entry排到首位
  if (lm.accessOrder) {
      lm.modCount++;
      // 先移除目标entry自身
      before.after = after;
      after.before = before;
      //插入到header之前
      addBefore(lm.header);
  }
}
// LRU算法，把当前entry插入到header队首之前，成为新的队首
private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
    after  = existingEntry;
    before = existingEntry.before;
    before.after = this;
    after.before = this;
}
```

首先调用HashMap的`getEntry`方法，根据key查到`entry`，然后通过`recordAccess`方法把查到的entry调整到队首位置，让他最靠前！！！



#### LinkedHashMap的总结

- 原理：以HashMap来维护数据结构，以LinkedList来记录插入顺序；
- 插入和查询处理：以内部entry的`addBefore`方法提供lru算法，把当前的目标entry排到队首，实现「entry越长时间不操作，插入顺序就越靠后」；
- 核心容器：父类HashMap中的table三列表，和自己维护的header双向链。

### LruCache的实现

#### 构造方法

```java

private final LinkedHashMap<K, V> map;

public LruCache(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```

设置了最大容量，并且初始化了一个`LinkedHashMap`。

#### put与get方法(基本可不看)

put方法：

```java
public final V put(K key, V value) {

    V previous;
    synchronized (this) {
        putCount++;
        size += safeSizeOf(key, value);
        //实际上就是调用LinkedHashMap的put
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, value);
    }

    // 回收机制在次
    trimToSize(maxSize);
    return previous;
}
```

get方法：

```java
public final V get(K key) {

    V mapValue;
    synchronized (this) {
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }

    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    synchronized (this) {
        createCount++;
        mapValue = map.put(key, createdValue);

        if (mapValue != null) {
            // There was a conflict so undo that last put
            map.put(key, mapValue);
        } else {
            size += safeSizeOf(key, createdValue);
        }
    }

    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        // 回收机制在次
        trimToSize(maxSize);
        return createdValue;
    }
}
```

#### 用来回收的trimToSize方法

```java
public void trimToSize(int maxSize) {
    // 永动机
    while (true) {
        K key;
        V value;
        synchronized (this) {

            if (size <= maxSize) {
                break;
            }

            // 拿到最老的entry
            Map.Entry<K, V> toEvict = map.eldest();

            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();

            // 移除最老的家伙
            map.remove(key);
            size -= safeSizeOf(key, value);
            evictionCount++;
        }

        // 单纯的回调
        entryRemoved(true, key, value, null);
    }
}
```

从linkedHashMap中，循环移除最老的entry，直到当前size小于maxSize。

### 总结LruCache的原理

内部维护了一个`LinkedHashMap`，可以设置最大容量，每次put和get时，都会循环移除linkedHashMap的队尾，直到小于maxSize。