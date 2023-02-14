---
title: Android内存泄露专题
date: 2021-03-24 16:24:04
tags: [Android, 内存泄露 ,NPE]
---

## 原因

短生命周期对象持有长生命周期对象的强引用，造成短生命周期对象在不需要使用时不能被回收。

### 静态变量导致

```java
public class MainActivity extends AppCompatActivity {

    public static boolean RESULT;

    private static Activity mActivity;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mActivity = this;
    }
}
```

### LeakCanary原理

```java
class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState) {

            }

            @Override
            public void onActivityStarted(@NonNull Activity activity) {

            }

            @Override
            public void onActivityResumed(@NonNull Activity activity) {

            }

            @Override
            public void onActivityPaused(@NonNull Activity activity) {

            }

            @Override
            public void onActivityStopped(@NonNull Activity activity) {

            }

            @Override
            public void onActivitySaveInstanceState(@NonNull Activity activity, @NonNull Bundle outState) {

            }

            @Override
            public void onActivityDestroyed(@NonNull Activity activity) {

            }
        });

    }
}
```

`Application`提供了`registerActivityLifecycleCallbacks`方法，可以监听Activity的生命周期，而`FragmentManager`中提供了`registerFragmentLifecycleCallbacks`方法可以监听Fragment的生命周期。

在`onActivityDestroyed`方法中监听到Activity正在销毁，就用弱引用去包装这个Activity，然后放入队列里面，从线程池中obtain出一个线程，5秒之后判断这个Activity的弱引用是否为null，如果为null就是已被回收了。如果不是null，说明没有被回收，可能存在内存泄露，这时候强制GC回收他，如果还不为null，说明确实存在内存泄露。

然后通过HAHA获取内存中heap堆快找引用关系。
