# 第五课：使用java定义系统服务.md

这是我跟随秋少的系列课程Android系统开发入门做的笔记，感谢原博主秋少！
[课程连接](http://qiushao.net/categories/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/)
<!--more-->

## 自定义流程

### 定义AIDL

创建`frameworks/base/core/java/android/veecar`目录，在其中添加`IHelloService.aidl`：

```
package android.veecar;

interface IHelloService {
    void hello(String name);
}
```

然后在`frameworks/base/Android.bp`的`framework`模块中添加AIDL的路径：

```
"core/java/android/veecar/IHelloService.aidl",
```

然后可以单编`frameworks/base`目录：

```
croot
mmm frameworks/base -j24
```

编译完成后会在`out/soong/.intermediates/frameworks/base/framework/android_common/gen/aidl/frameworks/base/core/java/android/veecar`下生成两个文件：
```
├── IHelloService.java
└── IHelloService.java.d
```

**其中`IHelloService.java`文件封装了binder通信的细节。**

### 实现接口

在`frameworks/base/services/core/java/com/android/server`目录下新建HelloService.java:

```java
package com.android.server;

import android.pure.IHelloService;
import android.util.Log;

public class HelloService extends IHelloService.Stub {
    private final String TAG = "HelloService";

    public HelloService() {
        Log.d(TAG, "create hello service");
    }

    @Override
    public void hello(String name) {
        Log.d(TAG, "hello " + name);
    }
}
```

### 注册Service

下一步就是在`SystemServer`中注册刚刚实现的Service。

修改 frameworks/base/services/java/com/android/server/SystemServer.java 文件，在 startOtherServices 方法里面增加以下代码：

```java
// add hello service
traceBeginAndSlog("HelloService");
ServiceManager.addService("HelloService", new HelloService());
traceEnd();
```

然后更新API接口：

```
make update_api
```

最后就可以先编译运行一下模拟器了：

```
m -j24
```

启动模拟器后，可以通过如下指令检测服务是否启动：

```shell
adb root
adb shell

service list | grep HelloService
```

如果服务正常启动了，会收到如下结果：
```
HelloService: [android.veecar.IHelloService]
```

### 遇到问题

按照秋少的教程实际编译是成功的，但是启动模拟器后，通过`service list`指令并没有找到我的Service，查询log后，发现如下错误：

```
03-27 15:56:56.050  1496  1496 E SELinux : avc:  denied  { add } for service=HelloService pid=1779 uid=1000 scontext=u:r:system_server:s0 tcontext=u:object_r:default_android_service:s0 tclass=service_manager permissive=0
```

### 添加Selinux权限

在`system/sepolicy`中通过下面的指令查找出所有与`network_time_update_service`相关的权限设置：

```
grep -nr network_time_update_service
```

然后在所有的`service_contexts`上添加：

```
HelloService                              u:object_r:HelloService:s0
```

在所有的`service.te `添加：

```
type HelloService, system_server_service, service_manager_type;
```

然后再次编译验证即可。

## 客户端验证

### 系统预置App模块直接调用

这个我按照秋少的教程没有成功（可能因为他是安卓10而我是安卓9），最后编译的时候报错，留待后面找时间再学习一下。 [  ]

### 编译出jar供外部App调用

#### 封装接口

创建目录如下：

```
~/myproject/AOSP/device/vee/veecar/models$ tree HelloApi/
HelloApi/
├── Android.bp
└── java
    └── com
        └── veecar
            └── api
                └── HelloManager.java
```

Android.bp:

```
java_library {
    name: "com.veecar.api",
    installable: true,
    srcs: [
        "java/**/*.java", //文件列表
    ],
}
```

HelloManager.java:

```
package com.veecar.api;

import android.os.RemoteException;
import android.os.ServiceManager;
import android.veecar.IHelloService;

public class HelloManager {

    private static HelloManager mInstance = null;
    public static HelloManager getInstance() {
        if (null == mInstance) {
            mInstance = new HelloManager();
        }
        return mInstance;
    }

    private IHelloService mService = null;
    private HelloManager() {
        mService = IHelloService.Stub.asInterface(ServiceManager.getService("HelloService"));
    }

    public void sayHello(String name) {
        try {
            mService.hello(name);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```

然后单编HelloApi模块，得到`out/target/common/obj/JAVA_LIBRARIES/com.veecar.api_intermediates/classes.jar`,然后写个App，调用：

```java
HelloManager.getInstance().sayHello("qiushao");
```

接下来还是要配置一下SeLinux权限,首先查找所有的`untrusted_app.te`：

```
~/myproject/AOSP/system/sepolicy$ find -name untrusted_app.te

./public/untrusted_app.te
./private/untrusted_app.te
./prebuilts/api/26.0/public/untrusted_app.te
./prebuilts/api/26.0/private/untrusted_app.te
./prebuilts/api/28.0/public/untrusted_app.te
./prebuilts/api/28.0/private/untrusted_app.te
./prebuilts/api/27.0/public/untrusted_app.te
./prebuilts/api/27.0/private/untrusted_app.te
```

在所有的private/untrusted_app.te中添加：

```
allow platform_app HelloService:service_manager find;
```

然后重新编译运行即可。
