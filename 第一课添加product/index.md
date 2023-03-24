# 第一课：添加product

这是我跟随秋少的系列课程Android系统开发入门做的笔记，感谢原博主秋少！
[课程连接](http://qiushao.net/categories/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/)
<!--more-->

## 前言

当我们编译AOSP时，需要先执行如下指令：

```
source build/envsetup.sh
lunch aosp_x86_64-eng
m -j24
```

其中`lunch`指令就是选择我们要编译的product。一般做产品时，我们会添加自己的product。

## 创建product的四要素

- vendorsetup.sh:把product添加到lunch选项
- BoardConfig.mk : 芯片硬件相关配置，分区设置等
- AndroidProducts.mk : 指定 product 配置
- product.mk : 一个产品的软件相关的配置，比如内置哪些软件模块，由AndroidProducts.mk 中的PRODUCT_MAKEFILES指定

- device:用来给厂商放product的。
- build/target目录，用来放google官方内置的product的。

## 创建device目录

device的目录结构：

```
device/[company]/[product]
```

新建我们的自己的目录：

```
mkdir -p device/vee/veecar
```

### 板子配置BoardConfig.mk

`BoardConfig.mk`包含了硬件芯片架构配置，分区大小等配置信息。

在device/vee/veecar下面，新建板子配置文件BoardConfig.mk，内容可以直接引用其他product的文件：

```
include $(SRC_TARGET_DIR)/board/generic_x86_64/BoardConfig.mk
```

其中`SRC_TARGET_DIR`指代`$(TOPDIR)build/target`,可在build/core/config.mk中找到定义。

### 产品配置[product].mk

新建产品配置文件veecar.mk内容如下：

```
$(call inherit-product, $(SRC_TARGET_DIR)/product/aosp_x86_64.mk)

PRODUCT_NAME   := veecar
PRODUCT_DEVICE := veecar
```

注意：

- PRODUCT_NAME要保持与文件名一致，编译输出目录就是由这个变量决定的。
- PRODUCT_DEVICE 跟BoardConfig.mk相关，编译系统会根据$(PRODUCT_DEVICE) 来加载对应目录下的BoardConfig.mk文件。

### 编译配置AndroidProducts.mk

新建AndroidProducts.mk文件：

```
PRODUCT_MAKEFILES := \
    $(LOCAL_DIR)/veecar.mk

// 从安卓10开始用来取代vendorsetup.sh的
COMMON_LUNCH_CHOICES := \
    veecar-eng
```

其中`PRODUCT_MAKEFILES`用来指定我们上一步中定义的product.mk文件，其中『veecar』对应我们的productName，eng则为build type:

- eng:对应到工程板。编译打包所有模块。同时ro.secure=0,ro.debuggable=1,ro.kernel.android.checkjni=1,表示adbd处于ROOT状态，所有调试开关都打开。
- userDebug:对应到用户调试版，同时ro.debuggable=1,打开调试开关，但没有开放ROOT权限。
- user：对应到用户版。同时ro.secure=1，ro.debuggable=0，关闭调试开关，关闭ROOT权限。最终发布到用户手里的都是user版。

### vendorsetup.sh

创建vendorsetup.sh文件如下：

```
add_lunch_combo veecar-eng
```

主要作用就是向lunch界面中添加有一个combo，当我们执行`source build/envsetup`后再`lunch`，就会发现多了一个『veecar-eng』项目。

**注意：从android10开始不需要这玩意了，使用`COMMON_LUNCH_CHOICES`来替代。

## 编译验证


执行下述指令编译验证。

```
source build/envsetup
lunch veecar-eng
m -j26
emulator
```
