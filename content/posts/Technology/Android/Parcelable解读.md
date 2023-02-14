---
title: Parcelable解读
date: 2021-03-12 14:43:10
tags: [Android]
---

```java
class Data implements Parcelable {

    /**
     * 字段
     */
    private String name;

    /**
     * 构造方法内读取
     * @param in
     */
    protected Data(Parcel in) {
        name = in.readString();
    }

    /**
     * 写入
     */
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
    }

    /**
     * 生成器常量
     */
    public static final Creator<Data> CREATOR = new Creator<Data>() {

        /**
         * 调用构造方法生成一个实例
         * @param in
         * @return
         */
        @Override
        public Data createFromParcel(Parcel in) {
            return new Data(in);
        }

        /**
         * 生成对应类型的数组
         * @param size
         * @return
         */
        @Override
        public Data[] newArray(int size) {
            return new Data[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }



    //===========以下是getter和setter访问器

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

`Parcelable`使用：

1. 继承`Parcelable`接口；
2. 声明构造方法，从`Parcel`中序列化数据；
3. 重写`writeToParcel(Parcel dest, int flags)`方法，用来反序列化；
4. 创建`Creator<Data>`类型的`CREATOR`，其内部实现了反序列化对象和数组的方法；
5. 重写`describeContents()`方法。

## Parcel

我们知道，序列化和反序列化都离不开`Parcel`对象，那他是个啥？

序列化的调用顺序：

```java
//在我们的自定义类中：
@Override
public void writeToParcel(Parcel dest, int flags) {
    dest.writeString(name);
}

//最终调用Parcel类中：
public void writeString16NoHelper(@Nullable String val) {
    nativeWriteString16(mNativePtr, val);
}

//本地JNI方法
@FastNative
private static native void nativeWriteString16(long nativePtr, String val);
```

Parcel用来包装数据，并且提供了大量的read/write方法，最终会调用本地方法**通过`Binder`读写数据到一块共享内存**。

这里就与`Serializable`有很大的不同。**`Serializable`的原理是通过IO将数据写到一个文件中**。

那么而这对比，肯定内存的读写性能要高于硬盘文件系统啊，这就是为什么Parcelable比Serializable快很多的原因。


## Parcel的创建

```java
//池子内最大6个
private static final int POOL_SIZE = 6;
private static final Parcel[] sOwnedPool = new Parcel[POOL_SIZE];
private static final Parcel[] sHolderPool = new Parce[POOL_SIZE];

@NonNull
public static Parcel obtain() {
    final Parcel[] pool = sOwnedPool;
    synchronized (pool) {
        Parcel p;
        for (int i=0; i<POOL_SIZE; i++) {
            p = pool[i];
            if (p != null) {
                //池子中取出一个
                pool[i] = null;
                //重置属性
                p.mReadWriteHelper = ReadWriteHelper.DEFAULT;
                //返回
                return p;
            }
        }
    }
    //如果池子是空的则new一个
    return new Parcel(0);
}
```

## Parcel的回收

```java
public final void recycle() {
    if (DEBUG_RECYCLE) mStack = null;
    freeBuffer();

    final Parcel[] pool;
    if (mOwnsNativeParcelObject) {
        pool = sOwnedPool;
    } else {
        mNativePtr = 0;
        pool = sHolderPool;
    }

    synchronized (pool) {
        for (int i=0; i<POOL_SIZE; i++) {
            if (pool[i] == null) {
                pool[i] = this;
                return;
            }
        }
    }
}
```

啥意思呢？本来这个Parcel是自由的，有个民警同志带着这个parcel去看守所转了一圈，一旦发现有空牢房，就把Parcel扔进去，就算回收了。等刑满社会需要Parcel的时候再放出来给社会做贡献。

## Parcel的销毁

```java
@Override
protected void finalize() throws Throwable {
    if (DEBUG_RECYCLE) {
        if (mStack != null) {
            Log.w(TAG, "Client did not call Parcel.recycle()", mStack);
        }
    }
    destroy();
}

private void destroy() {
    resetSqaushingState();
    if (mNativePtr != 0) {
        if (mOwnsNativeParcelObject) {
            nativeDestroy(mNativePtr);
            updateNativeSize(0);
        }
        mNativePtr = 0;
    }
}
```

竟然是重写`finalize()`方法手动销毁的！

**`finalize()`方法会在GC时回调，用来释放非Java对象，比如JNI中的C/C++对象**。此处是为了调用nativeDestroy(mNativePtr)方法释放本地C++对象。
