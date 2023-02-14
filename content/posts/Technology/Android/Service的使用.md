---
title: "Service的使用"
subtitle: ""
date: 2022-03-28T21:49:50+08:00
lastmod: 2022-03-28T21:49:50+08:00
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

## startService的生命周期

1. onCreate
2. onStartCommant
3. onDestroy

```kotlin
fun start(v: View) {
    Intent(this, MyService::class.java).run {
        startService(this)
    }
}

fun stop(v: View) {
    Intent(this, MyService::class.java).run {
        stopService(this)
    }

    // or 在service中调用stopSelf()
}
```

这种方式创建Service，可以多次调用`startService`，并且`onStartCommant`方法也会随之回调，但是`onCreate`只会回调一次。

可以通过Intent向service传递信息。

当不需要service不需要继续运行时，需要手动关闭。

## bindService的生命周期

1. onCreate
2. onBind
3. onDestroy

Activity代码：

```kotlin 
private lateinit var service: MyService

/**
  *  自定义一个连接，用来获取service对象
  */
private val conn = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, binder: IBinder) {
        // 获取service
        val myBinder  = (binder as MyService.MyBinder)
        //调用业务方法
        myBinder.startDownload.invoke()
    }

    override fun onServiceDisconnected(name: ComponentName?) {
        // 当Service未正常退出，而是因为其他原因退出时回调
    }
}

fun bind(v: View) {
    Intent(this, MyService::class.java).run {
        bindService(this, conn, BIND_AUTO_CREATE)
    }
}

fun unbind(v: View) {
    Intent(this, MyService::class.java).run {
        unbindService(conn)
    }
}
```

Service代码：

```kotlin
class MyService : Service() {

    /**
    * 预置一个自定义的binder
    **/
    private val mBinder = MyBinder {
        startDownload()
    }

    override fun onCreate() {
        Log.d(TAG, "onCreate: ")
        super.onCreate()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        Log.d(TAG, "onStartCommand: ")
        return super.onStartCommand(intent, flags, startId)
    }

    override fun onBind(intent: Intent): IBinder {
        // 返回自定义的binder
        return mBinder
    }

    override fun onDestroy() {
        Log.d(TAG, "onDestroy: ")
        super.onDestroy()

    }

    /**
     * Activity可以直接通过service对象调用业务方法
     */
    private fun startDownload() {
        //处理下载任务，完成后通知Activity刷新UI
        mBinder.callback?.refreshUI()

    }

    /**
     * 自定义一个Binder内部类
     * 通过传递函数参数，实现Activity直接通过binder调用service的内部方法
     */
    inner class MyBinder(val startDownload: () -> Unit) : Binder() {

        var callback: MyBinderCallback? = null
    }
}

interface MyBinderCallback {
    fun refreshUI()
}
```

- 通过`bindService`方法绑定service，可以使用`BIND_AUTO_CREATE`这个flag自动创建service，也可以直接绑定到`startService`方法创建的service之上。
- 当不再需要service继续执行时（例如Activity退出前），必须调用`unbindService`进行解绑，否则会报错。
- 当多个页面都绑定到同一个service，单个页面的解绑不会影响service。
- 当没有页面继续绑定时，service会自动销毁。
- 当通过`startService`创建了service，并且有其他activity又通过`bindService`绑定到了service，那么只有`unbindService`和`stopService`都调用之后，才会销毁service。