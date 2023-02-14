---
title: "SELinux和SEAndroid"
subtitle: ""
date: 2022-12-12T10:18:18+08:00
lastmod: 2022-12-12T10:18:18+08:00
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

### 模式

- 宽容模式-Permissive:只打印audit异常log，不会拒绝请求；
- 强制模式-Enforce:即打印audit异常log，也会拒绝请求；

### 策略文件

#### 位置

SELinux策略文件用于定义域和标签，以`*.te`结尾。

Android自带的策略文件位于AOSP中的`system/sepolicy`中，我们尽量**不要去修改这里的策略**，而是应修改位于`device/manufacturer/device-name/sepolicy`目录下（其实就是croot/device/制造商/设备/sepolicy/）的.te文件，同时也**尽量不要去新建.te文件，而是应该在原有的基础上修改**。

>其实这里说的尽量不要，主要是针对企业，因为google有可能会找你麻烦，而且后续更新也会遇到问题，但是如果自己玩的话，想改就改。不过还是不建议改默认实现，否则升级之后又回退了。

#### 格式

```
allow source target:class permissions;
```

- source:规则主题的类型（或属性），代表谁正在申请访问权限；
- target:对象类型（或属性），对哪些内容提出了访问权限；
- class:要访问的对象（）的类型;
- permissions:要执行的操作。

eg:

```
allow untrusted_app app_data_file:file {read write}; 
```

#### 宏

如果要添加的权限很多，为了简化规则定义，SEAndroid提供了宏，比如上面的例子可以这样写：

eg:

```
allow untrusted_app app_data_file:file rw_file_perms; 
```

### 上下文描述文件

在`device/manufacturer/device-name/sepolicy`中存在各种上下文描述文件，主要**负责为target打标签**。

#### 列举

- `file_contexts`用于为文件分配标签，并且可供多种用户空间时用。
- `genfs_contexts`用于为不支持扩展属性的文件系统分配标签。
- `property_contexts`用于为Android系统属性分配标签，以便控制那些进程可以为设置这些属性。再启动期间，`init`进程会读取这些配置。
- `service_contexts`用于为Android Binder服务分配标签，以便控制那些进程可以为相应服务添加（注册）和查找Binder引用。在启动期间，`servicemanager`进程会读取此配置。
- `seapp_contexts`用于为应用进程和`/data/data`目录分配标签。在每次应用启动时，`zygote`进程都会读取此配置；在启动期间，`installd`会读取此配置。
- `mac_permissions.xml`用于根据应用签名和应用软件包名称为应用分配`seinfo`标记。分配的`seinfo`标记可在`seapp_contexts`文件中用做密钥，以便为带有该`seinfo`标记的所有应用分配特定标签。在启动期间，`sestem_server`会读取此配置。


#### 格式

我们打开file_contexts文件，查看一下格式：

```
/mnt/vendor/crypto(/.*)?                            u:object_r:mnt_vendor_crypto_file:s0
/(vendor|system/vendor)/bin/delete7dayslog.sh       u:object_r:delete_seven_day_log-sh_exec:s0
```

就是**目录+标签**的方式，看起来很像host的定义。

这里要注意的是：

- 目录支持正则表达式
- u，object_r,s0都是固定的

### BoardConfig.mk

当修改或添加了策略文件和上下文描述文件后，需要更新`/device/manufacturer/device-name/BoardConfig.mk`来引用生效。

eg:

```
BOARD_SEPOLICY_DIRS += \
        <root>/device/manufacturer/device-name/sepolicy

BOARD_SEPOLICY_UNION += \
        genfs_contexts \
        file_contexts \
        sepolicy.te
```

**最后需要需要重新编译才能生效！**


### 启动用SELinux

1. 在内核中启用（eg:在kernel/msm-5.4/kernel/configs/android-base.config）中配置：`CONFIG_SECURITY_SELINUX=y`;
2. 更改Kernel_cmdline参数：`BOARD_KERNEL_CMDLINE := androidboot.selinux=permissive`,发版前应该移除此参数以变更为强制模式；

### 查看开机时的拒绝事件

```shell
adb root
adb shell dmesg | grep audit | grep av
```

返回：

```
[    1.979517] audit: type=1400 audit(1.969:4): avc:  denied  { relabelfrom } for  pid=261 comm="init" name="/" dev="tmpfs" ino=10968 scontext=u:r:vendor_init:s0 tcontext=u:object_r:tmpfs:s0 tclass=dir permissive=0
```

### 查看SContext

标准的SContext格式如下：

```
user:role:type[:range]
```

- User:用户，这里的用户并非linux的UID;
- Role：角色，不同的Role具有不同的权限；
- Type:Subject或者Object的类型，对于进程来说，type就是domain。
- Range：MSL级别。MLS将系统的进程和文件进行了分级，不同级别的资源需要对应级别的进程才能访问。

#### 文件的SContext

命令：`ls -Z`

```
 u:object_r:system_file:s0         system
```

- u:该文件的SELinux用户,在SEAndroid中只有一个SELinux用户，就是u；
- object_r:角色，这里的object_r表示固定不变的东西，如文件，在SEAndroid中只定义了两个role，适用于主题的`r`和适用于对象的`object_r`；
- system_file:类型，这里表示system目录的类型；
- s0:MLS的敏感度，默认就是`s0`。

#### 进程的SContext

命令：`ps -Z`

```
LABEL          USER        PID   PPID    VSZ     RSS     WCHAN       ADDR S NAME                       
u:r:su:s0      root       25497  8935    9320    3144   sigsuspend      0 S sh
u:r:su:s0      root       27256  25497   11788   3128   0               0 R ps
```

最左侧LABEL这一列就是进程的`SContext`：

- u:就是SELinux用户（user）；
- r:为角色（role），相当于用户组；
- su:代表该进程所属的**域**为SU;
- s0:MLS级别。


==================================================to be continue========