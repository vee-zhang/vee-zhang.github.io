---
title: "Android启动优化"
subtitle: ""
date: 2021-08-03T10:27:09+08:00
lastmod: 2021-08-03T10:27:09+08:00
draft: false
authors: []
description: ""

tags: [Android,性能优化]
categories: []
series: [Android性能优化]

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

### 回顾启动流程

1. 点击App图标；
2. Launcher进程通过Binder向SystemServer进程发起`startActivity`请求；
3. SystemServer收到请求后向zygote发起`创建进程`请求；
4. zygote收到请求fork自己创建新的子进程（App进程）；
5. App进程通过binder向SystemServer进程发起`attachApplication`请求；
6. SystemServer收到请求后，进行一系列操作后，通过Binder向App进程发送`scheduleLauncherActivity`请求；
7. App进程收到请求后，通过handler向主线程发送`LAUNCH_ACTIVITY`消息；
8. 主线程收到消息后，通过反射创建目标Activity，并调用`onCreate`方法。

### 应用启动状态

- 冷启动：是指应用从头开始启动，系统进程在冷启动后才创建应用进程；
- 热启动：系统的所有工作就是将Activity带到前台，不需要重新创建、加载和绘制；
- 温启动： 短时间内重启应用，进程可能未被销毁，直接继续运行，但是需要重走生命周期。

我们一般**针对冷启动进行优化**。

### 应用启动时间

是指从点击到launcher，经过Application到第一个Activity加载渲染完毕的总时间。

所以优化的点有两个：

- Application的处理速度
- MainActivity的加载速度

### 统计启动时间

1. 通过logcat中对display关键字检索，在启动后有个Displayed+包名+时间段；
    ```shell
    ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms
    ```
2. 通过adb命令：
    ```shell
    adb shell am start -S -W [packageName]/{activityName}
    ```
    输出：
    ```shell
    ThisTime: 415 # 启动所有Activity的耗时
    TotalTime: 415 # 表示冷启动的耗时，不包括前一个Activity的pause耗时
    WaitTime: 437 #总的耗时，包括前一个Activity的pause耗时和新Activity的启动耗时
    ```
3. 通过Profile工具中的CPU性能剖析器。
    - Call Chart 倒火焰图；
    - flame Chart 火焰图；
    - TopDown 详情试图。

4. Debug-API：
    ```kotlin
    Debug.startMethodTracing()
    Debug.stopMethodTracing()
    ```

5. 通过配置严苛模式

### 黑白屏处理

为MainActivity自定义一个style，设置背景图，如下：

```xml
<style name="Theme.App.NoActionBar.Launcher">
    <item name="android:windowBackground">@drawable/header</item>
</style>
```

然后在MainActivity的`onCreate`方法中把主题再设置回去：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        setTheme(R.style.Theme_App_NoActionBar)
        super.onCreate(savedInstanceState)
        setContent {
            NewsStory()
        }
    }
}
```

> AsyncLayoutInflater异步加载