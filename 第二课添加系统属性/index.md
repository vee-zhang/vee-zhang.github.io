# 第一课：添加系统属性

这是我跟随秋少的系列课程Android系统开发入门做的笔记，感谢原博主秋少！
[课程连接](http://qiushao.net/categories/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/)
<!--more-->

## 前言

之前死记硬背过，在安卓的启动过程中，init.rc会加载系统属性，现在也不知道记得对不对。如果是这样的话，可想系统属性是启动的非常早的，非常适合做一些初始化变量的存放。

在车端开发中，系统启动时可从配置字读取数据，存放到系统属性，供fw和上层应用读取数值，依据不同的数值加载不同的配置或者操作。

用途非常强大！


## 系统属性

可以简单的理解为`key-value`形式的系统层级环境变量。key和value都只能是字符串。

## 前缀

- `ro`    只读属性
- `persist`   持久化属性，重启不会失效，保存在/data/property目录
- `ctrl`  用来启动和停止服务。init一旦收到『设置strl.start属性的请求』，属性服务将使用该属性值作为服务名找到该服务并启动。

## 用法

adb :

```
# 读取
adb shell getprop key value

# 设置
adb shell setprop key value
```

java:

```java
// 读取系统属性
SystemProperties.get(String key, String def) 

// 设置系统属性
SystemProperties.set(String key, String val) 
```

C++:

```c++
#include <sys/system_properties.h>

int property_get(const char *key, char *value, const char *default_value)
{
    int len = __system_property_get(key, value);
    if (len > 0)
    {
        return len;
    }
    if (default_value)
    {
        len = strnlen(default_value, PROPERTY_VALUE_MAX - 1);
        memcpy(value, default_value, len);
        value[len] = '\0';
    }
    return len;
}
```

## 系统属性的配置文件

- /default.prop 或者是 /prop.default
- /vendor/default.prop
- /system/build.prop
- /vendor/build.prop
- /data/local.prop
- /data/property/*

系统会按照先后顺序依次加载以上文件，所以后加载的属性会覆盖原先的值。

其中，default.prop的值是通过build/tools目录下的buildinfo.sh 和 vendor_buildinfo.sh 生成的，要修改的话就得修改编译系统了，这种方式不好维护，不推荐修改。

## 添加系统属性

### 添加到/system/build.prop

只要在 $TARGET_DEVICE_DIR 目录创建一个 system.prop 文件，在里面添加属性即可。
编译系统会把 $(TARGET_DEVICE_DIR)/system.prop 添加到 /system/build.prop 文件中去。

创建`device/vee/veecar/system.prop`文件：

```
ro.vee.version=1.0
```

然后重新编译系统后，发现`out/target/product/system.build.prop`中找不到我们新添加的属性。

原因是**Android 9.0 之后，google不推荐把厂家定制的 property 加到 /system 分区了。**

解决方式：

在`device/vee/veecar/BoardConfig.mk`中指定system.prop文件：

```
TARGET_SYSTEM_PROP += device/vee/veecar/system.prop
```

再次编译后发现就有我们添加的系统属性了。

### 添加到/vendor/build.prop

在`device/vee/veecar/veecar.mk`中添加`PRODUCT_PROPERTY_OVERRIDES`变量：

```
PRODUCT_PROPERTY_OVERRIDES += \
	ro.auth.name=vee \
	persist.vendor.auth.name=vee \
	vendor.vee.name=vee
```

编译后，在`out/target/product/veecar/vendor/build.prop`文件中就能找到我们新定义的三个系统属性了。
