---
title: "ubuntu22.04上不能使用adb"
date: 2022-10-10T17:24:07+08:00
draft: false
---

今天重新安装了ubuntu20.04LTS，安装了IDEA后发现不能用adb连接车机，报错：

```
no permissions (missing udev rules? user is in the plugdev group)
```

在[这里](https://blog.csdn.net/weixin_43230861/article/details/119422383)找到了解决办法，感谢原作者！

方法如下：

### 查看所有USB设备信息

```
adb start-server
lsusb
```

结果：

```
// 原作者的：
$adb start-server
$lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub 
Bus 001 Device 003: ID 17ef:6099 Lenovo 
Bus 001 Device 002: ID 17ef:608d Lenovo 
Bus 001 Device 033: ID 18d1:4ee7 Google Inc. //android xxx
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

// 我的：
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 001 Device 002: ID 25a7:2420        2.4G Wireless Device
Bus 001 Device 008: ID 12d1:3a07 Huawei Technologies Co., Ltd. HUAWEI USB-C HEADSET
Bus 001 Device 007: ID 05c6:901d Qualcomm, Inc. 
Bus 001 Device 004: ID 214b:7250  USB2.0 HUB
Bus 001 Device 006: ID 12d1:107e Huawei Technologies Co., Ltd. EVR-AL00
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

这里就要根据如上信息找到自己设备的id，比如作者的应该是picxl手机，所以是【Google Inc】的那一行，而我的是车机，应该是高通的那一行【Qualcomm, Inc】。

### 创建配置文件

```
sudo vim /etc/udev/rules.d/90-android.rules
```

>这个文件名可以随意哈🦐

内容：

```
ATTRS{idVendor}=="自己设备那一行ID的左侧数字 我这里是18d1" 
ATTRS{idProduct}=="自己设备那一行ID的右侧数字 我这里是4ee7"
```

### 启用配置文件

```
sudo udevadm control --reload-rules
sudo service udev restart
sudo udevadm trigger
adb kill-server
adb devices
```

然后就ok了！