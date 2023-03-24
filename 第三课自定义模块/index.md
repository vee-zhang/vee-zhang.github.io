# 第三课：自定义模块

这是我跟随秋少的系列课程Android系统开发入门做的笔记，感谢原博主秋少！
[课程连接](http://qiushao.net/categories/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/)
<!--more-->

## 创建一个模块

为了方便理解，我们先上手创建一个模块。

### 添加源文件

在我们之前定义的product目录下创建一个模块目录`hello`，并在其中创建`hello.cpp`：

```c++
#include <cstdio>
#include <android/log.h>

#define LOG_TAG "qiushao"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG ,__VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,LOG_TAG ,__VA_ARGS__)

int main() {
    printf("hello vee\n");
    LOGD("hello vee");
    return 0;
}
```

### 添加必须的Android.bp文件

在同级目录创建Android.bp：

```
cc_binary {              //模块类型为可执行文件
    name: "hello",       //模块名hello
    srcs: ["hello.cpp"], //源文件列表
    vendor: true,        //编译出来放在/vendor目录下(默认是放在/system目录下)
    shared_libs: [       //编译依赖的动态库
        "liblog",
    ],
}
```

### 单编验证

这个模块就定义完成了，现在可以单编这个模块：

```
mmm device/vee/veecar/hello/
```

编译完成后就可以在out/target/product/veecar/vendor/bin目录下看到hello可执行文件了。

### 添加系统编译

刚才我们只是单编，是通过`mmm`告诉系统要编译这个模块，但如果整编的话，系统是不知道我们新添加了一个模块的。所以需要向系统注册模块。

在我们的/device/vee/veecar/veecar.mk中添加：

```
PRODUCT_PACKAGES += hello
```

然后整编系统就可以了。

### 模拟器验证

启动模拟器后：

```
adb shell hello

输出：

```

## 了解Android.bp

AOSP的编译工具是soong，这货用Android.bp文件来定义编译方式。

每一个模块（module）的根目录下都应该有一个Android.bp（或者已经过期的Android.mk）。

### 类型

Android.bp中也可以设置一些变量有如下类型：

- 布尔值（true or false）
- 整数(int)
- 字符串("string")
- 字符串列表（["string1","string2"]）
- 映射({key1:"value1",key2:["value2"]})

>映射可以包含任何类型的值，包括嵌套映射。列表和映射可能在最后一个值后面有终止逗号。——引自https://source.android.google.cn/docs/setup/build?hl=zh_cn

那我理解，一个模块除了模块类型之外的其他部分其实都是映射。

### 格式

```
cc_binary {
    name: "gzip",
    srcs: [
      "src/test/minigzip.c",
      "src/test/minigzip1.c"
    ],
    shared_libs: ["libz"],
    stl: "none",
}
```

上面的代码描述的就是一个模块。看起来很像json，但并不是json。

他以`module type`开头，后跟一组**映射**。

- `name`属性: 必须，表示模块名称，在所有模块中唯一；
- `srcs`：指定模块的源文件，也可以通过『模块引用语法』`:<module-name>`引用其他模块的输出。

### 变量

变量的声明类似于bash中的变量声明：

```
gzip_srcs = ["src/test/minigzip.c"],
```

变量是在声明后才开始生效的。

我们来看个例子：

```
gzip_srcs = ["src/test/minigzip.c"],
cc_binary {
    name: "gzip",
    srcs: gzip_srcs,//直接使用变量
    shared_libs: ["libz"],
    stl: "none",
}
```

**变量是不可变的。可以使用`+=`将变量附加到 别处。**

### 注释

- 单行注释：`//`
- 多行注释：`/*  */`

### 运算符

可以使用`+`运算符来：

- 拼接字符串、字符串列表和映射
- 整数求和

### 全局模式与通配符

接受文件的属性可以采用UNIX通配符`*`，如：

- `*.java`
- `java/**/*.java`同时匹配 `java/Main.java` 和 `java/com/android/Main.java`

### 条件编译

可以在Android.bp中为不同条件添加不同属性，具体的逻辑判断是在Go语言中实现的。

例如对于特定架构：

```
cc_library {
    ...
    srcs: ["generic.cpp"],
    arch: {
        arm: {
            srcs: ["arm.cpp"],
        },
        x86: {
            srcs: ["x86.cpp"],
        },
    },
}
```

### 格式化命令

```
bpfmt -w .
```

可以递归对当前目录和子目录中所有的Android.bp文件进行格式化，包括：

- 缩进4个空格
- 多元素列表的每个元素后面有换行符
- 列表和映射末尾有英文逗号

## 特殊模块

### 默认模块

`cc_defaults`可以封装一些公共属性，提供给其他模块使用：

```
cc_defaults {
    name: "gzip_defaults",
    shared_libs: ["libz"],
    stl: "none",
}

cc_binary {
    name: "gzip",
    defaults: ["gzip_defaults"],
    srcs: ["src/test/minigzip.c"],
}
```

### 命名空间模块

Soong 可以让不同目录中的模块指定相同的名称，只要每个模块都在单独的命名空间中声明即可。可以按如下方式声明命名空间：

```
soong_namespace {
    imports: ["path/to/otherNamespace1", "path/to/otherNamespace2"],
}
```

**注意：`soong_namespace`模块不需要`name`属性，他的所在路径会自动成为模块name。**

### 预构建模块



### 具体定义

具体定义在`build/soong/androidmk/cmd/androidmk/android.go`。

```
var moduleTypes = map[string]string{
	"BUILD_SHARED_LIBRARY":        "cc_library_shared",
	"BUILD_STATIC_LIBRARY":        "cc_library_static",
	"BUILD_HOST_SHARED_LIBRARY":   "cc_library_host_shared",
	"BUILD_HOST_STATIC_LIBRARY":   "cc_library_host_static",
	"BUILD_HEADER_LIBRARY":        "cc_library_headers",
	"BUILD_EXECUTABLE":            "cc_binary",
	"BUILD_HOST_EXECUTABLE":       "cc_binary_host",
	"BUILD_NATIVE_TEST":           "cc_test",
	"BUILD_HOST_NATIVE_TEST":      "cc_test_host",
	"BUILD_NATIVE_BENCHMARK":      "cc_benchmark",
	"BUILD_HOST_NATIVE_BENCHMARK": "cc_benchmark_host",

	"BUILD_JAVA_LIBRARY":             "java_library",
	"BUILD_STATIC_JAVA_LIBRARY":      "java_library_static",
	"BUILD_HOST_JAVA_LIBRARY":        "java_library_host",
	"BUILD_HOST_DALVIK_JAVA_LIBRARY": "java_library_host_dalvik",
	"BUILD_PACKAGE":                  "android_app",
}
```

|module type|description|
|---|---|
|cc_library_shared|编译成动态库|
|cc_library_static|编译成静态库|
|cc_library_host_shared||
|cc_library_host_static||
|cc_library_headers||
|cc_binary|编译成可执行文件|
|cc_binary_host||
|cc_test||
|cc_test_host||
|cc_benchmark||
|cc_benchmark_host||
|java_library|构建.jar，如果指定“installable:true”将生成一个包含“classes.dex”文件的“.jar”文件，适合在设备上安装|
|java_library_static|同上，已过时|
|java_library_host||
|java_library_host_dalvik||
|android_app|构建apk|

## 模块输出分区

- system :主要包含 Android 框架， google 官方实现
    - Android.mk 默认就是输出到 system 分区，不用指定
    - Android.bp 默认就是输出到 system 分区，不用指定
- vendor :SoC芯片商分区(系统级核心厂商，如高通), 为他们提供一些核心功能和服务，由 soc 实现
    - Android.mk LOCAL_VENDOR_MODULE := true
    - Android.bp vendor: true
- odm :设备制造商分区（如华为、小米），为他们的传感器或外围设备提供一些核心功能和服务
    - Android.mk LOCAL_ODM_MODULE := true
    - Android.bp device_specific: true
- product :产品机型分区
    - Android.mk LOCAL_PRODUCT_MODULE := true
    - Android.bp product_specific
