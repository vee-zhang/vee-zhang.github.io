---
title: "手把手教你自定义HAL层"
subtitle: ""
date: 2022-08-03T17:30:15+08:00
lastmod: 2022-08-03T17:30:15+08:00
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
最近工作要求，涉及到了HAL层开发，让我这个刚刚从上层转底层的小白无所适从，根本不知道怎么入手，从网上搜集了大堆的文章，照着做没有一个能成，前前后后一个月了，今天终于成功了，特此记录一下，希望能帮到其他像我一样高转低的同行吧。
<!--more-->

>本文大量借鉴[这篇文章](https://www.codenong.com/cs109234795/)，非常感谢原作！

## 环境介绍

- aosp版本：android9
- 电脑操作系统：ubuntu20.04LTS
- 手机：google-pixcel-3xl-欧版

## 编译过程

### 定义软件包文件结构

我们定义的一部分相关功能的集合，叫做**软件包**。

软件包名称可以具有子集，如`package.subpackage`，其根目录为`hardware/interfaces`或`vendor/vendorName`。软件包名称，在根目录下形成一个或多个子目录。

AOSP中的软件包分布如下：

| 前缀                 | 位置                             | 类型             |
| -------------------- | -------------------------------- | ---------------- |
| android.hardware.*   | hardware/interfaces/*            | HAL              |
| android.frameworks.* | frameworks/hardware/interfaces/* | frameworks/ 相关 |
| android.system.*     | system/hardware/interfaces/*     | system/ 相关     |
| android.hidl.*       | system/libhidl/transport/*       | 核心             |

从上表看到，我们的HAL应该位于hardware/interfaces/*中，但由于AOSP某些分支是锁定的，不允许修改VNDK库列表，所以我们不能直接修改`hardware/interfaces/*`中的文件，一个很好的解决方式就是在我们的厂商目录`vendor`中建立同样的`hardware/interfaces/*`，然后映射到AOSP根目录下的相同位置。这种类型的HAL叫做**厂商扩展**。


接下来我们就定义一个软件包的目录结构：

```shell
mkdir -p vendor/sample/hardware/interfaces/helloworld/1.0
```

我们在`vendor`中创建hal模块，仔细研究目录结构：

- sample是我们的组织名称；
- hardware/interfaces映射着着根目录里面的hardware/interfaces；
- interfaces后面将作为我们的root目录；
- helloworld是模块名称；
- 1.0是一个版本目录，以后可能还有2.0,3.0。

>网上很多文章都是直接在根目录下的hardware/interface中操作，我试了很多次都编译失败了，都是关于VNDK的报错，VNDK是啥我都不知道，也没办法解决。后来才知道，不要直接在Android源码目录hardware/interfaces下面添加和修改接口,在API锁定分支中不允许更改VNDK库列表。

### 创建HIDL文件

接下来，我们在`1.0`目录中创建三个`.hal`文件：

#### IHelloWorld.hal
   
```
package sample.hardware.helloworld@1.0;

import IHelloWorldCallback;

interface IHelloWorld {

    initial();
    getInt() generates (int32_t i);
    setInt(int32_t val) generates (Result error);
    oneway setCallback(IHelloWorldCallback callback);

};
```

写法非常类似`AIDL`，事实上，**从Android11开始，google已经用AIDL去替代HIDL**,所以对于HIDL大概理解就行，没必要下功夫去学。

内容不多，我们在此声明了四个虚方法，后面会一一实现。

#### IHelloWorldCallback.hal

```
package sample.hardware.helloworld@1.0;

interface IHelloWorldCallback {

  oneway onEvent(Event event);

};
```

#### types.hal

`types.hal`文件并没有定义接口，而是用来定义软件包中每个接口可以访问的数据类型。根据Google的规范，这个文件的名称必须是types.hal。

```
package sample.hardware.helloworld@1.0;

enum Result : int32_t {
    OK,
    UNKNOWN,
    INVALID_ARGUMENTS,
};

struct Event {
    uint32_t type;
    uint32_t code;
    uint32_t value;
};
```

最后注意一下`package`包声明，是与目录有一定联系的！

### 指定interfaces为软件包根目录

在interfaces中创建`Android.bp`文件：

```
hidl_package_root {
    name: "sample.hardware",
    path: "vendor/sample/hardware/interfaces",
}
```

我们知道`.bp`文件是用来描述软件包的，这里就是把interfaces指定为“sample.hardware”软件包的根目录。

>如果没有指定目录，编译时会报错：Cannot find package root specification

### 用hidl-gen工具生成相关文件

`hidl-gen`是google提供的，用来生成hidl相关文件的工具，但他本身是以源码的形式，放在aosp代码中提供的，所以使用前需要先编译这玩意：

```
cd到android9的根目录

source build/envsetup.sh
lunch 20
make hidl-gen -j4
```

为了方便，我们可以先定义几个环境变量，一会要重复使用：

```shell
// 包名
PACKAGE=sample.hardware

// 软件名
NAME=$PACKAGE.helloworld@1.0

// 软件包根目录
ROOT=vendor/sample/hardware/interfaces

// hidl实现文件存放的位置
LOC=$ROOT/helloworld/1.0/default
```

### 生成版本控制文件

版本控制文件是位于软件包根目录下的`current.txt`,用来做OTA升级时，确认软件包版本信息的。

```shell
hidl-gen -Lhash -r$PACKAGE:$ROOT -randroid.hidl:system/libhidl/transport $NAME> $ROOT/current.txt
```

在$ROOT下会出现一个current.txt文件，里面是所有hidl的哈希，用来标记版本。

### 生成HIDL的实现

```shell
hidl-gen -o $LOC -Lc++-impl -r$PACKAGE:$ROOT -randroid.hidl:system/libhidl/transport $NAME
```

在$LOC下会生成HIDL的c++实现文件，我们需要修改成自己的实现。

#### HelloWorld.h

```c++
#ifndef SAMPLE_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H
#define SAMPLE_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H

#include <sample/hardware/helloworld/1.0/IHelloWorld.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace sample {
namespace hardware {
namespace helloworld {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct HelloWorld : public IHelloWorld
{
    Return<void> initial() override;
    Return<int32_t> getInt() override;
    Return<::sample::hardware::helloworld::V1_0::Result> setInt(int32_t val) override;
    Return<void> setCallback(const sp<::sample::hardware::helloworld::V1_0::IHelloWorldCallback> &callback) override;

    static void *pollThreadWrapper(void *me);
    void pollThreadEntry();

private:
    pthread_mutex_t mLock = PTHREAD_MUTEX_INITIALIZER;
    pthread_t mPollThread;
    bool mRunning;
    std::vector<sp<IHelloWorldCallback>> mCallbacks;
};

}  // namespace implementation
}  // namespace V1_0
}  // namespace helloworld
}  // namespace hardware
}  // namespace sample

#endif  // SAMPLE_HARDWARE_HELLOWORLD_V1_0_HELLOWORLD_H
```

#### HeloWorld.cpp:

```c++
#define LOG_TAG "sample.hardware.helloworld@1.0-service"
#include <android-base/logging.h>
#include <log/log.h>

#include "HelloWorld.h"

namespace sample {
namespace hardware {
namespace helloworld {
namespace V1_0 {
namespace implementation {

Return<void> HelloWorld::initial()
{
    ALOGD("%s", __FUNCTION__);

    if (mRunning)
        return Void();

    mRunning = true;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
    pthread_create(&mPollThread, &attr, pollThreadWrapper, this);
    pthread_attr_destroy(&attr);

    return Void();
}

Return<int32_t> HelloWorld::getInt()
{
    ALOGD("%s", __FUNCTION__);
    return int32_t{666};
}

Return<::sample::hardware::helloworld::V1_0::Result> HelloWorld::setInt(int32_t val)
{
    ALOGD("%s %d", __FUNCTION__, val);
    return ::sample::hardware::helloworld::V1_0::Result{};
}

void *HelloWorld::pollThreadWrapper(void *me)
{
    static_cast<HelloWorld *>(me)->pollThreadEntry();
    return NULL;
}

void HelloWorld::pollThreadEntry()
{
    mRunning = true;
    ALOGD("%s enter", __FUNCTION__);

    Event event;
    event.type = 0;
    event.code = 0;
    event.value = 0;
    while (mRunning)
    {
        event.code++;
        event.type++;
        event.value++;
        sleep(1);
        ALOGD("%s event %04x %04x %04x\n", __FUNCTION__, event.type, event.code, event.value);
        pthread_mutex_lock(&mLock);
        if (!mCallbacks.empty())
        {
            std::vector<sp<IHelloWorldCallback>>::iterator it;
            for (it = mCallbacks.begin(); it != mCallbacks.end();)
            {
                sp<IHelloWorldCallback> callback = *it;
                Return<void> ret = callback->onEvent(event);
                if (!ret.isOk())
                {
                    ALOGE("error %s", ret.description().c_str());
                    it = mCallbacks.erase(it);
                }
                else
                {
                    it++;
                }
            }
        }
        pthread_mutex_unlock(&mLock);
    }
    ALOGD("%s exit", __FUNCTION__);
}

Return<void> HelloWorld::setCallback(const sp<::sample::hardware::helloworld::V1_0::IHelloWorldCallback> &callback)
{
    ALOGD("%s", __FUNCTION__);
    pthread_mutex_lock(&mLock);

    if (callback != nullptr)
    {
        mCallbacks.push_back(callback);
    }

    pthread_mutex_unlock(&mLock);
    return Void();
}

}  // namespace implementation
}  // namespace V1_0
}  // namespace helloworld
}  // namespace hardware
}  // namespace sample
```

#### service.cpp

以上全都是c++代码，c++需要一个`main`入口函数才能启动，接下来就要实现一个service作为入口，创建`$LOC/service.cpp`:

```c++
#define LOG_TAG "sample.hardware.helloworld@1.0-service"

#include <android-base/logging.h>
#include <hidl/HidlTransportSupport.h>

#include "HelloWorld.h"

using android::OK;
using android::sp;
using android::status_t;
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;
using sample::hardware::helloworld::V1_0::IHelloWorld;
using sample::hardware::helloworld::V1_0::implementation::HelloWorld;

int main(int /* argc */, char ** /* argv */)
{
    ALOGD("HAL Service is starting.");
    sp<IHelloWorld> service = new HelloWorld();

    configureRpcThreadpool(1, true /*callerWillJoin*/);
    status_t status = service->registerAsService();
    if (status != OK)
    {
        LOG_ALWAYS_FATAL("Could not register service for HelloWorld HAL Iface (%d).", status);
        return -1;
    }

    ALOGD("Register as service ready.");

    joinRpcThreadpool();
    return 1; // joinRpcThreadpool shouldn't exit
}
```

逻辑很简单，创建了`main`入口函数，在其中new了一个HelloWorld类，并调用其`registerAsService()`函数。

然后需要把service添加进Android.bp，才会被编译，修改后的`$LOC/Android.bp`：

```
cc_library_shared {
    name: "sample.hardware.helloworld@1.0-impl",

    // 要把proprietary: true改为vendor: true
    vendor: true,
    // 删除下面这一行
    //relative_install_path: "hw",
    srcs: [
        "HelloWorld.cpp",
    ],
    shared_libs: [
        "liblog",
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "sample.hardware.helloworld@1.0",
    ],
}

cc_binary {
    name: "sample.hardware.helloworld@1.0-service",
    defaults: ["hidl_defaults"],
    vendor: true,
    relative_install_path: "hw",
    init_rc: ["sample.hardware.helloworld@1.0-service.rc"],
    srcs: ["service.cpp"],

    shared_libs: [
        "libcutils",
        "libhidlbase",
        "libhidltransport",
        "liblog",
        "libutils",
        "libhardware",
        "sample.hardware.helloworld@1.0",
        "sample.hardware.helloworld@1.0-impl",
    ],
}
```

>这里要注意，将cc_library_shared节点proprietary: true改为vendor: true;删除relative_install_path: “hw”,目的是将sample.hardware.helloworld@1.0-impl.so编译安装到vendor/lib64目录下面,如果vendor/lib64/hw下面,运行服务时因VNDK规则限制,会报错:F linker : CANNOT LINK EXECUTABLE “./vendor/bin/hw/sample.hardware.helloworld@1.0-service”: library “sample.hardware.helloworld@1.0-impl.so” not found

到这里我们就成功添加了程序入口，但是问题又来了，有了入口函数，系统怎么认识他呢？为了让系统认识他，我们需要添加init.rc文件，为系统做指引。

#### init.rc

创建`$LOC/$NAME@1.0-service.rc`文件：

```
service vendor.helloworld-1-0 /vendor/bin/hw/sample.hardware.helloworld@1.0-service
    class hal
    user system
    group system
```

init.rc文件的意义，就是把我们刚刚添加的程序入口，作为服务启动：

- 第一行，创建一个service？？？
- 第二行，service类型为hal；
- 第三行，用户为system
- 第四行，用户组为system

有了实现，但是系统并不会编译它，下一步就是生成Android.bp，指导系统如何编译。

### 生成Android.bp

生成$LOC下的Android.bp:

```shell
hidl-gen -o $LOC -Landroidbp-impl -r$PACKAGE:$ROOT -randroid.hidl:system/libhidl/transport $NAME
```

生成$ROOT/helloworld/1.0中的Android.bp：

```shell
$ source $ANDROID_BUILD_TOP/system/tools/hidl/update-makefiles-helper.sh
$ do_makefiles_update $PACKAGE:ROOT android.hardware:hardware/interfaces android.hidl:system/libhidl/transport
```

### 在编译类型中注册HAL服务

在`devices`中的对应厂商的对应编译类型（lunch指令的编译目标）目录下修改manifest.xml文件，比如我的aosp,对应调试手机为`google-pixel-3xl`,我在lunch时的目标类型是`aosp-crosshatch-userdebug`，那么就应该去修改`device/google/crosshatch/manifest.xml`文件。在文件中加入：

```xml
<hal format="hidl">
    <name>sample.hardware.helloworld</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>IHelloWorld</name>
        <instance>default</instance>
    </interface>
</hal>
```

然后整编一下，更新manifest.xml。

>这里不更新manifest.xml,可能会报错：W hwservicemanager: getTransport: Cannot find entry sample.hardware.helloworld@1.0::IHelloWorld/default in either framework or device manifest.

在这里我遇到了报错：

```shell
[ 25% 4/16] build out/target/product/crosshatch/gen/ETC/framework_compatibility_matrix.xml_intermediates/compatibility_matrix.xml
FAILED: out/target/product/crosshatch/gen/ETC/framework_compatibility_matrix.xml_intermediates/compatibility_matrix.xml 
/bin/bash -c "PRODUCT_ENFORCE_VINTF_MANIFEST=\"true\"           VINTF_ENFORCE_NO_UNUSED_HALS=true               out/host/linux-x86/bin/assemble_vintf           -i out/target/product/crosshatch/system/etc/vintf/compatibility_matrix.legacy.xml:out/target/product/crosshatch/system/etc/vintf/compatibility_matrix.1.xml:out/target/product/crosshatch/system/etc/vintf/compatibility_matrix.2.xml:out/target/product/crosshatch/system/etc/vintf/compatibility_matrix.3.xml:out/target/product/crosshatch/system/etc/vintf/compatibility_matrix.device.xml            -o out/target/product/crosshatch/gen/ETC/framework_compatibility_matrix.xml_intermediates/compatibility_matrix.xml             -c \"out/target/product/crosshatch/obj/ETC/device_manifest.xml_intermediates/manifest.xml\""
Error: The following instances are in the device manifest but not specified in framework compatibility matrix: 
    sample.hardware.helloworld@1.0::IHelloWorld/default
Suggested fix:
1. Check for any typos in device manifest or framework compatibility matrices with FCM version >= 3.
2. Add them to any framework compatibility matrix with FCM version >= 3 where applicable.
3. Add them to DEVICE_FRAMEWORK_COMPATIBILITY_MATRIX_FILE.
ninja: build stopped: subcommand failed.
10:01:12 ninja failed with: exit status 1

#### failed to build some targets (7 seconds) ####
```

猜想原因，是因为由于锁定分支的关系，我们没有直接在`hardware/interfaces/*`中建立我们的hal,而是在厂商目录`vendor`中玩的。

注意看错误信息：`Error: The following instances are in the device manifest but not specified in framework compatibility matrix: 
    sample.hardware.helloworld@1.0::IHelloWorld/default`。意思是说，我们虽然在device的manifest中添加了一个实例，但是没有在framework compatibility matrix中指定。

然后下面给出了3条建议：

```shell
1. Check for any typos in device manifest or framework compatibility matrices with FCM version >= 3.
2. Add them to any framework compatibility matrix with FCM version >= 3 where applicable.
3. Add them to DEVICE_FRAMEWORK_COMPATIBILITY_MATRIX_FILE.
```

然后根据建议，在`hardware/interfaces/compatibility_matrices/compatibility_matrix.3.xml`中添加相同的内容：

```xml
<hal format="hidl">
    <name>sample.hardware.helloworld</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>IHelloWorld</name>
        <instance>default</instance>
    </interface>
</hal>
```

再次整编成功！

- [ ] 后面需要详细了解一下[兼容性矩阵](https://source.android.google.cn/devices/architecture/vintf/comp-matrices)。

### 添加本地c++测试


// 包名
PACKAGE=sample.hardware

// 软件名
NAME=$PACKAGE.helloworld@1.0

// 软件包根目录
ROOT=vendor/sample/hardware/interfaces

// hidl实现文件存放的位置
LOC=$ROOT/helloworld/1.0/default


创建`$ROOT/1.0/test/Android.cpp`：

```
cc_binary {
    name: "sample.hardware.helloworld_hidl_hal_test",
    vendor: true,
    srcs: ["test.cpp"],

    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "liblog",
        "libutils",
        "libhardware",
        "sample.hardware.helloworld@1.0",
    ],
}
```

创建`$ROOT/1.0/test/test.cpp`:

```c++
#define LOG_TAG "sample.hardware.helloworld_hidl_hal_test"

#include <android-base/logging.h>
#include <hidl/HidlTransportSupport.h>
#include <log/log.h>
#include <sample/hardware/helloworld/1.0/IHelloWorld.h>
#include <sample/hardware/helloworld/1.0/types.h>

using ::android::sp;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::sample::hardware::helloworld::V1_0::Event;
using ::sample::hardware::helloworld::V1_0::IHelloWorld;
using ::sample::hardware::helloworld::V1_0::IHelloWorldCallback;
using ::sample::hardware::helloworld::V1_0::Result;

class HelloWorldCallback : public IHelloWorldCallback
{
public:
    HelloWorldCallback()
    {
        printf("%s\n", __FUNCTION__);
    }
    ~HelloWorldCallback()
    {
        printf("%s\n", __FUNCTION__);
    }

    Return<void> onEvent(const Event &event)
    {
        printf("%s %04x %04x %04x\n", __FUNCTION__, event.type, event.code, event.value);
        return Void();
    }
};

int main(int /* argc */, char ** /* argv */)
{
    sp<IHelloWorld> service = IHelloWorld::getService();
    if (service == nullptr)
    {
        printf("Failed to get service\n");
        return -1;
    }

    printf("initial\n");
    service->initial();

    int ret = service->getInt();
    printf("getInt %d\n", ret);

    Result result = service->setInt(888);
    printf("setInt result=%d\n", result);

    sp<IHelloWorldCallback> callback = new HelloWorldCallback();
    printf("setCallback\n");
    service->setCallback(callback);
    while (1)
    {
        sleep(1);
    }

    return 0;
}
```

### 编译出运行库

>因为我是小白，还不能充分理解，下面完全摘抄原作。

```shell
mmm vendor/sample/hardware/interfaces/helloworld/1.0/
```

编译hal文件生成的中间代码位于out/soong/.intermediates/vendor/sample/hardware/interfaces/helloworld/1.0/:

```
default
sample.hardware.helloworld@1.0
sample.hardware.helloworld@1.0-adapter
sample.hardware.helloworld@1.0-adapter_genc++
sample.hardware.helloworld@1.0-adapter-helper
sample.hardware.helloworld@1.0-adapter-helper_genc++
sample.hardware.helloworld@1.0-adapter-helper_genc++_headers
sample.hardware.helloworld@1.0_genc++
sample.hardware.helloworld@1.0_genc++_headers
sample.hardware.helloworld-V1.0-java
sample.hardware.helloworld-V1.0-java_gen_java
test
```

"sample.hardware.helloworld@1.0_genc++"目录里面可以发现hal文件转换成了C++源码 ,里面就包含Binder Bn端,Binder Bp端的代码.不用像非Project Treble项目那样需要自己写Binder Bn端,Binder Bp端的代码.体会到了谷歌的良苦用心。

编译最终会在out/target/product/$TARGET_PRODUCT/目录下面生成了以下文件:

```
out/target/product/$TARGET_PRODUCT/system/lib/sample.hardware.helloworld@1.0.so
out/target/product/$TARGET_PRODUCT/system/lib/sample.hardware.helloworld@1.0-adapter-helper.so
out/target/product/$TARGET_PRODUCT/system/lib64/sample.hardware.helloworld@1.0.so
out/target/product/$TARGET_PRODUCT/system/lib64/sample.hardware.helloworld@1.0-adapter-helper.so
out/target/product/$TARGET_PRODUCT/system/framework/sample.hardware.helloworld-V1.0-java.jar
out/target/product/$TARGET_PRODUCT/system/framework/oat/arm/sample.hardware.helloworld-V1.0-java.odex
out/target/product/$TARGET_PRODUCT/system/framework/oat/arm/sample.hardware.helloworld-V1.0-java.vdex
out/target/product/$TARGET_PRODUCT/system/framework/oat/arm64/sample.hardware.helloworld-V1.0-java.odex
out/target/product/$TARGET_PRODUCT/system/framework/oat/arm64/sample.hardware.helloworld-V1.0-java.vdex
out/target/product/$TARGET_PRODUCT/vendor/lib/sample.hardware.helloworld@1.0-impl.so
out/target/product/$TARGET_PRODUCT/vendor/lib/sample.hardware.helloworld@1.0.so
out/target/product/$TARGET_PRODUCT/vendor/lib/sample.hardware.helloworld@1.0-adapter-helper.so
out/target/product/$TARGET_PRODUCT/vendor/lib64/sample.hardware.helloworld@1.0-impl.so
out/target/product/$TARGET_PRODUCT/vendor/lib64/sample.hardware.helloworld@1.0.so
out/target/product/$TARGET_PRODUCT/vendor/lib64/sample.hardware.helloworld@1.0-adapter-helper.so
out/target/product/$TARGET_PRODUCT/vendor/bin/hw/sample.hardware.helloworld@1.0-service
out/target/product/$TARGET_PRODUCT/vendor/bin/sample.hardware.helloworld_hidl_hal_test
out/target/product/$TARGET_PRODUCT/vendor/etc/init/sample.hardware.helloworld@1.0-service.rc
```

## 测试

### 推送包到设备

将编译好的库和bin文件复制到设备中:

```shell
$ adb root
$ adb remount
$ adb push  out/target/product/$TARGET_PRODUCT/vendor/lib64/sample.hardware.helloworld@1.0-impl.so /vendor/lib64/sample.hardware.helloworld@1.0-impl.so
$ adb push  out/target/product/$TARGET_PRODUCT/vendor/lib64/sample.hardware.helloworld@1.0.so /vendor/lib64/sample.hardware.helloworld@1.0.so
$ adb push  out/target/product/$TARGET_PRODUCT/vendor/bin/hw/sample.hardware.helloworld@1.0-service /vendor/bin/hw/sample.hardware.helloworld@1.0-service
$ adb push  out/target/product/$TARGET_PRODUCT/vendor/bin/sample.hardware.helloworld_hidl_hal_test /vendor/bin/sample.hardware.helloworld_hidl_hal_test
```

>我尝试直接整编，然后刷机，后续操作直接失败，说是找不到服务，然后进手机对应的目录，确实没有发现对应的库，暂时不知道是为什么。所以还是需要自己push到设备上。

- [ ] 整编刷机的问题

### 启动服务

在adb中启动服务：

```shell
# ./vendor/bin/hw/sample.hardware.helloworld@1.0-service&
```

注意：

- 指令最后的「&」符号表示后台运行。
- 不加「&」符号，输入指令后，shell会卡住不动，是正常的。
- 输入指令后，不会有任何输出，是正常的，想看输出，需要从logcat中查看。

### 启动测试

```shell
# sample.hardware.helloworld_hidl_hal_test
```

>这里我又遇到问题：Cannot find entry in either framework or device manifest

解决方式：在设备的`system/etc/vintf/manifest.xml`和`vendor/etc/vintf/manifest.xml`这两个文件中添加：

```xml
<hal format="hidl">
    <name>sample.hardware.helloworld</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>IHelloWorld</name>
        <instance>default</instance>
    </interface>
</hal>
```

再次尝试调用客户端成功！输出如下：

```shell
initial
getInt 666
setInt result=0
HelloWorldCallback
setCallback
onEvent 0001 0001 0001
onEvent 0002 0002 0002
onEvent 0003 0003 0003
...
```

### Java层App调试

新建一个Android应用项目，然后将`out/soong/.intermediates/vendor/sample/hardware/interfaces/helloworld/1.0/sample.hardware.helloworld-V1.0-java/android_common/combined/sample.hardware.helloworld-V1.0-java.jar`复制到app的`libs`目录下，右键选择`add as library`，之后app的gradle:

```groovy
implementation files('libs\\sample.hardware.helloworld-V1.0-java.jar')
```

MainActivity.kt:

```kotlin
package site.leasa.haltest

import android.os.Bundle
import android.os.RemoteException
import android.util.Log
import android.view.View
import androidx.appcompat.app.AppCompatActivity
import sample.hardware.helloworld.V1_0.Event
import sample.hardware.helloworld.V1_0.IHelloWorld
import sample.hardware.helloworld.V1_0.IHelloWorldCallback

class MainActivity : AppCompatActivity() {

    private lateinit var service: IHelloWorld

    private val TAG = "HAL_SERVICE_ACTIVITY"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }

    open fun click(v: View) {
        try {
            this.service = IHelloWorld.getService()
            service.initial()
            val ret: Int = service.getInt()
            Log.d(TAG, "getInt=$ret")
            service.setInt(888)
            service.setCallback(HelloWorldCallback())
        } catch (e: RemoteException) {
            Log.e(TAG, "hal服务初始化失败: ", e)
        }
    }
}

class HelloWorldCallback : IHelloWorldCallback.Stub() {

    private val TAG = "HAL_SERVICE_Callback"

    override fun onEvent(event: Event) {
        Log.d(TAG, "onEvent type=" + event.type + ", code=" + event.code + ", value=" + event.value);
    }

}
```

然后控制台进adb shell,启动服务：

```shell
# ./vendor/bin/hw/sample.hardware.helloworld@1.0-service&
```

启动App,点一下按钮出发`click方法`后，会报错：`NoSuchElement`，就是找不到我们的hal服务，之所以出现这个问题，是因为我们没有配置`selinux`的标签，这玩意太麻烦了，放后面学习，现在先关掉selinux：

```shell
adb shell setenforce 0
```

然后重启启动app,点击按钮，logcat控制台输出如下：

```shell
initial
getInt 666
setInt result=0
HelloWorldCallback
setCallback
onEvent 0001 0001 0001
onEvent 0002 0002 0002
onEvent 0003 0003 0003
...
```

## 后记

对于hal,我目前只是最粗浅的能够编译并运转，里面很多机制还不了解，后续会慢慢梳理好，补充道本文，这里还是特别感谢[这篇文章](https://www.codenong.com/cs109234795/)，对我的学习产生莫大帮助！

安卓系统是个很复杂的东西，当然HAL层启到承上启下的桥梁作用，对于我这个新手来说上手难度较大，本文的东西也是断断续续用了很长时间，磕磕绊绊解决了很多问题才产生的结果，可以说不容易，很庆幸，同时也见识到了操作系统层面的复杂精深，对知识广度有很高的要求。攻克下来，一马平川！还是那句话：If i want, i can do it!