---
title: ThreadLocal解读
date: 2021-03-17 10:59:56
tags:
---

## 从Set方法入手

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

从这里可以看出，threadLocal的核心其实是`ThreadLocalMap`对象。

`set`方法的作用是初始化内部的`ThreadLocalMap`，并将元素放入`ThreadLocalMap`中暂存，当前的ThreadLocal对象为key。

然后关注`createMap()`方法：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

每个线程中都有一个`threadLocals`对象，每次调用`createMap()`方法时都会给线程的`threadLocals`创建一个新的`ThreadLocalMap`，那么每次调用`set()`方法，其实是set给当前线程的`ThreadLocalMap`。

结论：`ThreadLocal`的`set`方法其实是把值存到了当前线程内部的`threadLocals`，而`threadLocals`是一个`ThreadLocalMap`类型的map（map嵌套），key就是当前的`ThreadLocal`。


## Get

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

//ThreadLocal内部方法
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

## ThreadLocalMap

ThreadLocalMap其实是个简化的HashMap，内部是个`Entry`类型的数组。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

## 真的线程安全？

大部分博客都说`ThreadLocal`是把变量在每个线程生成变量副本，线程只能操作属于自己的副本，从而不会相互影响，并且线程安全。

但是我翻来覆去的看代码，也没有看到打破引用的代码啊，没有`clone`，没有`new`，那怎么能叫副本呢？那应该不管怎么赋值，都是共享变量的引用才对啊。那么在不同线程中操作同一共享变量，又没加锁，还谈什么线程安全？

那就写个例子来印证一下：

```java
public class MainActivity extends AppCompatActivity {

    //随便创建一个线程池
    private final ThreadPoolExecutor tpe = new ThreadPoolExecutor(8, 8, 1, TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>());

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    //点击按钮后让线程池执行两个任务
    Data data;
    public void start(View v) {
        data = new Data();
        tpe.execute(new MyRunnable1(data));
        tpe.execute(new MyRunnable2(data));
    }

    //等10秒之后点第二个按钮展示结果
    public void toastResult(View v) {
        Log.d("我试试", "toastResult: " + data);
        Toast.makeText(this, "toastResult: " + data, Toast.LENGTH_SHORT).show();
    }

    //任务1
    static class MyRunnable1 implements Runnable {

        private final ThreadLocal<Data> mThreadLocal = new ThreadLocal<>();

        Data data;

        MyRunnable1(Data data) {
            this.data = data;
        }

        @Override
        public void run() {
            mThreadLocal.set(data);
            mThreadLocal.get().name = "李四";
            mThreadLocal.get().age = 10;
            try {
                Thread.sleep(10*1000);
                Log.d("我试试", "线程1： " + data);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    // 任务2 
    static class MyRunnable2 implements Runnable {

        private ThreadLocal<Data> mThreadLocal = new ThreadLocal<>();


        Data data;

        MyRunnable2(Data data) {
            this.data = data;
        }

        @Override
        public void run() {
            try {
                Thread.sleep(3000);
                Log.d("我试试", "线程2：" + data);
                mThreadLocal.set(data);
                mThreadLocal.get().name = "王五";
                mThreadLocal.get().age = 20;
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    static class Data {
        int age = 1;
        String name = "张三";

        Data(int age, String name) {
            this.age = age;
            this.name = name;
        }

        Data() {
        }

        @NonNull
        @Override
        public String toString() {
            return this.name + this.age;
        }
    }
}
```

上面代码的逻辑：创建一个变量，然后分别在两个线程中用set进各自的ThreadLocal，然后get各自的变量副本，修改值，打印当前值。最后等个20秒以后点一下获取结果按钮。

但是当我最后获取结果的时候，发现原始的Data数据变了，变成了任务2中的赋值结果！而且在两个任务执行期间，`threadLocal.get()`也能看到其他变量对data的修改结果，果然跟我的猜测不错！

通过印证，我发现ThreadLocal并不能解决线程安全问题，也并不是什么共享变量的副本。他的作用只是把数据与线程绑定，而且通过弱引用的方式避免线程作用域内的对象内存泄露。

## 为什么Looper采用ThreadLocal？

因为一般我们操作线程的变量，都要在线程里面去改它。我们可能new一个Thread或者new一个Runnable，通过构造方法或者在run方法里面去写变量赋值。那么问题来了，如果你拿不到线程，不能修改线程呢？比如ActivityThread。

而Looper的源码里面没有一句操作线程的代码，而是巧用了ThreadLocal完成了Looper自己与线程的绑定！！！

所以ThreadLocal的应用场景是：**不需要在线程中做操作，只需要把变量与线程绑定时就用ThreadLocal!ThreadLocal就是个中介者模式！**
