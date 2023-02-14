---
title: "Android.mk与ndk-build"
subtitle: ""
date: 2022-10-28T14:05:44+08:00
lastmod: 2022-10-28T14:05:44+08:00
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

## 编译方式

- 基于Make的ndk-build
- CMake
- 独立工具链

## ndk-build方式

ndk-build是基于make的编译工具，需要通过Android.mk和Application.mk来配置。

### 内部原理

运行`ndk-build`脚本相当于运行了以下命令：

```shell
$GNUMAKE -f <ndk>/build/core/build-local.mk
<parameters>
```

`$GNUMAKE` 指向 GNU Make 3.81 或更高版本，<ndk> 则指向 NDK 安装目录。您可以根据这些信息从其他 Shell 脚本（甚至是您自己的 Make 文件）中调用 ndk-build。

### 使用

```shell
$ cd <project>
$ <ndk安装目录>/ndk-build [option]
```

| option                    | mean                                                               |
| ------------------------- | ------------------------------------------------------------------ |
| clean                     | 清除之前的产物                                                     |
| V=1                       | 启动构建                                                           |
| -B V=1                    | 强行执行完整的构建并显示                                           |
| NDK_APPLICATION_MK=<file> | 使用 NDK_APPLICATION_MK 变量指向的特定 Application.mk 文件进行构建 |
| -C <project>              | 指定项目路径                                                       |

## Android.mk讲解

>之前学习AOSP的时候，google说这货已经废弃了，在aosp中用Android.bp逐渐替代。但是我一直找不到bp的相关文档，现在才只知道bp文件里面那些shared_library这些都是直接从Android.mk中继承来的，我不学mk，就不会写bp！现在学习ndk时发现在AS创建项目时默认使用CMake，而CMake用的是CMakeLists.txt，我去你XX的。

### 用途

1. 配置项目和指定Application.mk文件。
2. 模块化，`Android.mk`的语法支持将源文件**分组为模块**，**模块是静态库、动态库或独立的可执行文件**。
>因为Application.mk是在Android.mk中指定的，所以ndk-build中不需要指定前者，而需要指定后者，再通过后者找到前者。

### 基础

我从ndk目录中直接打开了一个Android.mk文件如下：

```
# 必须先定义这个变量，表示源文件在项目树中的位置，my-dir是ndk提供的宏，返回当前路径
LOCAL_PATH:= $(call my-dir)

$(warning ndk_helper is no longer maintained in the NDK. This copy is left for \
          compatibility purposes only. For an up to date copy, see \
          https://github.com/googlesamples/android-ndk/tree/master/teapots/common/ndk_helper)

# CLEAR_VARS指针指向一个特殊的GNU Makefile，可以清除LOCAL_XXX变量
include $(CLEAR_VARS)

# 指定模块的名称，每个模块名称唯一且不含空格，产物以这个值来命名
LOCAL_MODULE:= ndk_helper变量

# 列举源文件，以空格分割
LOCAL_SRC_FILES:= JNIHelper.cpp interpolator.cpp tapCamera.cpp gestureDetector.cpp perfMonitor.cpp vecmath.cpp GLContext.cpp shader.cpp gl3stub.c

# 变量
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)
# 变量
LOCAL_EXPORT_LDLIBS    := -llog -landroid -lEGL -lGLESv2
# 变量
LOCAL_STATIC_LIBRARIES := cpufeatures android_native_app_glue

# BUILD_STATIC_LIBRARY变量指向一个Makefile脚本，用来收集最近include以来在LOCAL_XXX变量中定义的信息
include $(BUILD_STATIC_LIBRARY)

#$(call import-module,android/native_app_glue)
#$(call import-module,android/cpufeatures)

```

## 变量

变量分为：

- ndk内置变量：以LOCAL_、PRIVATE_、NDK_、APP_开头的和部分小写字母变量；
- 自定义变量，推荐以MY_开头。

### NDK定义的include变量

#### CLEAR_VARS

用来取消所有LOCAL_XXX变量。

比如：LOCAL_MODULE、LOCAL_SRC_FILES 和 LOCAL_STATIC_LIBRARIES。
```
include $(CLEAR_VARS)
```

>不会取消LOCAL_PATH。

#### BUILD_EXECUTABLE

用于构建可执行文件。

```
include $(BUILD_EXECUTABLE)
```

需要：

- LOCAL_MODULE
- LOCAL_SRC_FILES

#### BUILD_SHARED_LIBRARY

**用于构建动态库.so**。

```
include $(BUILD_SHARED_LIBRARY)
```

需要：

- LOCAL_MODULE
- LOCAL_SRC_FILES

#### BUILD_STATIC_LIBRARY

用于编译静态库.a。

```
include $(BUILD_STATIC_LIBRARY)
```

#### PREBUILT_SHARED_LIBRARY

用于在编译动态库之前搞事情。

```
include $(PREBUILT_SHARED_LIBRARY)
```

>这里的`LOCAL_SRC_FILES`不能是源文件，而是预构建共享库的路径，如：foo/libfoo.so。

>也可以使用`LOCAL_PREBUILTS`变量引用另一个模块中的预构建库。

#### PREBUILT_STATIC_LIBRARY

同上，用来在编译静态库之前搞事情的。

### 目标信息变量

#### APP_ABI

我们可以在Android.mk中定义一个`APP_ABI`变量指定编译的ABI（一个或多个），构建系统会根据这货来解析Android.mk，指定的ABI月多4则解析次数也越多。

#### TARGET_ARCH

定义支持的cpu架构：

- arm
- arm64
- x86
- x86_64

#### TARGET_PLATFORM

定义支持的Android API级别号:

- android-1
- android-2
- android-3
- android-28

```
ifeq ($(TARGET_PLATFORM),android-22)
    # ... do something ...
endif
```

#### TARGET_ARCH_ABI

用来指定ABI：

| CPU架构       | ABI         |
| ------------- | ----------- |
| ARMv7         | armeabi-v7a |
| ARMv8 AArch64 | arm64-v8a   |
| i686          | x86         |
| x86-64        | x86_64      |

```
ifeq ($(TARGET_ARCH_ABI),arm64-v8a)
  # ... do something ...
endif
```

#### TARGET_ABI

用来指定特定设备，说白了就是`TARGET_PLATFORM`-`TARGET_ARCH_ABI`的形式：

```
ifeq ($(TARGET_ABI),android-22-arm64-v8a)
  # ... do something ...
endif
```

### 模块描述变量（重点）

#### LOCAL_PATH

用于指定当前文件的路径，**必须在Android.mk文件的开头定义此变量**。

```
LOCAL_PATH := $(call my-dir)
```

>不会被CLEAR_VARS指向的脚本清掉。

#### LOCAL_MODULE

模块名称。

```
LOCAL_MODULE := foo
LOCAL_MODULE_FILENAME := libnewfoo
```

产物默认使用模块名称来命名，如果需要单独命名产物，则可以使用`LOCAL_MODULE_FILENAME`变量。

上面代码中编译出来的产物就是libnewfoo.so。

#### LOCAL_SRC_FILES

用来指定构建编译的源文件列表，只需要列出构建时世纪传递到编译器的文件，构建系统会自动处理依赖关系，所以一般给这个变量指定一个入口文件就够了。

>推荐使用相对路径。

#### LOCAL_CPP_EXTENSION

可以使用此可选变量为 C++ 源文件指定 .cpp 以外的文件扩展名。

```
LOCAL_CPP_EXTENSION := .cxx
```

>没啥用。

#### LOCAL_CPP_FEATURES

太复杂，看不明白，后面需要再捋一下[文档](https://developer.android.google.cn/ndk/guides/android_mk?hl=zh_cn#local_cpp_features)


## 宏

宏就是NDK提供的内置函数。

可以使用`$(call <function>)`方式调用宏，返回字符串。

### my-dir

这个宏返回**最后**包括的 makefile 的路径，通常是当前 Android.mk 的目录。

但是有时候重复调用就会出问题，比如:

```
LOCAL_PATH := $(call my-dir)

# ... declare one module

# 你看这里，我include了一个也带有makefile的LOCAL_PATH
include $(LOCAL_PATH)/foo/`Android.mk`

# 再调用my-dir的话，路径就成了LOCAL—PATH的值了
LOCAL_PATH := $(call my-dir)

# 所以我们也可以在此include回原来的路径来解决这个问题🦐
include $(LOCAL_PATH)/foo/Android.mk

# ... declare another module
```

### all-subdir-makefiles

返回位于当前 my-dir 路径所有子目录中的 Android.mk 文件列表。

### this-makefile

返回当前 makefile（构建系统从中调用函数）的路径。

### parent-makefile

返回包含树中父 makefile 的路径（包含当前 makefile 的 makefile 的路径）。

### grand-parent-makefile

返回包含树中祖父 makefile 的路径（包含当前父 makefile 的 makefile 的路径）。

### import-module

用来导入模块（会自动导入目标模块的Android.mk）

```
$(call import-module,<name>)
```