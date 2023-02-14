---
title: "APK瘦身"
subtitle: ""
date: 2021-08-09T09:15:00+08:00
lastmod: 2021-08-09T09:15:00+08:00
draft: false
authors: []
description: ""

tags: [Android, 性能优化]
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

### APK文件的构成

- META-INF/：包含`CERT.SF`和`CERT.RSA`签名文件，和`MANIFEST.MF`清单文件；
- assets/：不需编译的资源文件；
- res/：包含需要编译到`resource.arsc`，但还未编译的资源；
- lib/：包含特定于处理器软件层的已编译代码armeabi等；
- resource.arsc：包含已编译的资源；
- classes.dex：我们的代码；
- AndroidManifest.xml：核心Android清单文件。

### 瘦身方法

#### 移除未使用资源

可以通过IDE提供的建议，和lint检查工具检测出未使用的资源、无用的import、字段、方法、类，然后删掉。

##### 如何保护必须资源

lint 工具不会扫描 assets/ 文件夹、通过反射引用的资源或已链接至应用的库文件。此外，它也不会移除资源，只会提醒您它们的存在。

那么有些资源暂时用不到，我们又需要保留的，可以通过创建一个包含`<resources>`的xml文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
tools:keep="@layout/l_used*_c,@layout/l_used_a,@layout/l_used_b*"
tools:discard="@layout/unused2" />
```

- tools:keep  指定每个要保留的资源；
- tools:discard   指定要舍弃的资源；

>这两个属性都接受逗号分隔符和*号通配符。

#### 启用资源缩减

- `minifyEnabled`，开启代码混淆；
- `shrinkResources`，开启资源缩减。

#### 指定本地化语言

如果使用了包含了语言的库（如AppCompat），那么打包的时候会把所有已翻译的语言的字符串都打包进去。

所以我们可以指定需要的语言来缩减apk体积：

```proovy
android {
  defaultConfig {
    ...
    resConfigs "zh-rCN"
  }
}
```

#### 指定需要的动态库so

```proovy
android{
  defaultConfig{
    ndk{
      abiFilters "armeabi-v7a"
    }
  }
}
```

armeabi-v7a兼容arm64架构，但是不如arm64的so性能高。我们可以配合`productFlavor`和`flavorDimensions`来分别针对arm和arm64设备来打包。

#### 图片瘦身

可以使用webp替代png和jpg，可以使用`VectorDrawable`矢量图替代应用图标。

可以通过ImageView的`tint`属性前台改色，达到复用图片的目的。

#### 开启本地库压缩

```xml
<application
        android:name="io.flutter.app.FlutterApplication"
        android:extractNativeLibs="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:networkSecurityConfig="@xml/network_security_config"
        android:requestLegacyExternalStorage="true"
        android:usesCleartextTraffic="true">
```

当`android:extractNativeLibs = true`时，gradle打包会对工程中的so进行压缩，使最终生成的apk体积变小。但用户在安装过程中，系统会对so库解压，从而造成apk安装时间变长。
