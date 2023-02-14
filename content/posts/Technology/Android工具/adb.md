---
title: "adb"
subtitle: ""
date: 2022-07-07T11:04:41+08:00
lastmod: 2022-07-07T11:04:41+08:00
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

## adb客户端环境变量

```
# ANDROID_HOME
export ANDROID_HOME=$ENV_HOME/Android

# android-sdk
export ANDROID_SDK_ROOT=${ANDROID_HOME}/sdk
export PATH=$PATH:$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/tools
```

把sdk中的`platform-tools`和`tools`加入PATH即可。

## adb的组成

- **客户端**：位于电脑，用于发送命令；
- **服务器**：位于电脑，用于管理客户端与守护进程之间的通信；
- **adbd**：位于设备，守护进程，用于在设备上运行命令。

执行过程：

1. 我在终端中，通过adb命令向客户端输入命令；
2. 客户端检查服务器是否运行，没有则启动服务器进程；
3. 服务器进程与本地TCP端口5037绑定，并监听客户端发出的命令，双方基于5037通信；
4. 服务器通过扫描设备的5555～5585之间的奇数号端口查找模拟器，发现adbd后建立连接；
5. 此时客户端可以通过adb命令控制设备。

## wifi-adb

### Android11及以上

1. 设备上，开发者模式，启动无线调试，选择`Pair device with pairing code`；记下设备的IP、端口号和配对码；
2. 运行`adb pair ipaddr:port`，出现提示时输入配对码。

### Android11以下

1. usb线连设；
2. 执行`adb tcpip 5555`来监听设备的5555；
3. 拔线
4. 执行`adb connect device_ip_addresss:5555`，其实很多情况下我都不加端口，默认就是5555；
5. 执行`adb devices`确认连接成功。

## 常用adb命令

- `adb kill-server` 杀掉adb服务器。
- `adb start-server` 启动adb服务器，一般不需要用。
- `adb devices -l` 查询设备,`-l`参数可打印出更多的信息。
- `adb -s <device serial>` 多个设备中指定一个设备。
- `adb -e` 多个设备中只有一个模拟器，则指定模拟器。
- `adb -d` 多个设备中只有一个硬件，则指定硬件。
- `adb connect <ip:port>` 连接一个设备。
- `adb disconnect` 断连所有设备。
- `adb install apk_file_path` 把apk安装到用户分区data/data。
- `adb uninstall <package name>` 通过包名卸载App。
- `adb forward tcp:6100 tcp:7100` 端口转发。
- `adb pull remote local` 拉取设备文件。
- `adb push local remote` 推送本地文件到设备，会覆盖设备同名文件。
- `adb --help`  调取帮助信息。
- `adb [-d|-e|-s] shell` 进入设备的shell。
- `adb [-d|-e|-s] shell <command>` 向设备发送shell指令。
- `exit` 或者快捷键`control+D`退出shell。
- `adb root` 同 `adb shell su` 启动管理员权限。
- `adb unroot` 取消管理员权限。
- `adb remount` 重新挂在system分区，system分区默认只读，remount之后变为读写权限。
- `adb disable-verity -R` 关闭分区检测，需要root权限，**重启生效**，`-R`表示自动重启。
- `adb reboot [bootloader|recovery|sideload|sideload-auto-reboot]` 设备重启。
- `adb logcat` 抓取log。

## adb shell
### ActivityManager(am)

#### 启动Activity 

```
start [options] intent 
```

| option                       | description                       |
| ---------------------------- | --------------------------------- |
| -D                           | 启用调试                          |
| -W                           | 等待启动完成                      |
| --start-profile \<filePath\> | 启动性能剖析器并将结果发送至file  |
| -P \<file\>                  | 同上，但当应用空闲时停止          |
| -R \<count\>                 | 启动Activity count次              |
| -S                           | 启动Activity前强行停止目标应用    |
| --opengl-trace               | 启用OpenGL汉书追踪                |
| --user \<userID\>\|currnet   | 指定作为哪个用户运行。默认current |

#### 启动Service

```
startservice [options] intent
```

| option                     | description                       |
| -------------------------- | --------------------------------- |
| --user \<userID\>\|currnet | 指定作为哪个用户运行。默认current |

#### 终止进程

```
force-stop [package] // 强制停止包名相关的所有进程

kill [--user userID|all|current] [package] //安全终止包名相关进程，可指定用户，默认all

kill-all //终止所有后台进程
```

#### 发送广播

```
broadcast [--user userID|all|current] intent
```

#### 性能监控

```
profile start [process] [file]//启动进程的性能剖析器，结果写入文件
profile stop [process]//停止性能剖析
```

#### 系统监视

```
monitor [--gdb] //开始监控崩溃或ANR,---gdb参数可在给定的端口上启动gdbserv
```

#### 调整显示

```
screen-compat {on | off} package	//控制 package 的屏幕兼容性模式。
display-size [reset | widthxheight] //替换设备显示尺寸
display-density dpi //替换设备显示密度
```

主要用于在大屏上调试小屏应用，或者反过来。

#### 查看intent规范

```
to-uri intent
to-intent-uri intent
```

### PackageManager(pm)

#### 列出软件包

```
list packages [options] [filter]
```

| option        | description        |
| ------------- | ------------------ |
| -f            | 查看关联文件       |
| -d            | 只显示已停用的包   |
| -e            | 只显示已启用的包   |
| -s            | 只显示系统应用     |
| -3            | 只显示第三方软件   |
| -i            | 查看软件包安装程序 |
| -u            | 包含已卸载的软件   |
| --user userID | 查询指定用户空间   |

#### 查看权限组

```
list permission-groups
list permissions [options] group
```

| option | description          |
| ------ | -------------------- |
| -g     | 按组整理             |
| -f     | 输出所有信息         |
| -s     | 输出摘要             |
| -d     | 列出危险权限         |
| -u     | 列出用户能看到的权限 |

#### 查找应用安装路径

```
path [package]
```

#### 安装软件

```
install [options] [path]
```

| option       | description              |
| ------------ | ------------------------ |
| -r           | 重装应用保留数据         |
| -t           | 允许安装debug包          |
| -d           | 允许降级安装             |
| -g           | 授予应用清单中的所有权限 |
| --fastdeploy | 只更新差异               |

#### 卸载软件

```
uninstall [-k] [package] //-k表示保留数据
clear [package] // 删除数据
```

### 截屏/录屏

```
screencap <filepath> // 截屏到文件

screenrecord [options] <filepath> // 录屏
```

可以通过`control+C`停止，或者通过`--time-limit`设置时间，默认3分钟。

## 附录：help信息

```
Android Debug Bridge version 1.0.41
Version 33.0.2-8557947
Installed as /home/vee/.env/Android/sdk/platform-tools/adb

global options:
 -a                       listen on all network interfaces, not just localhost
 -d                       use USB device (error if multiple devices connected)
 -e                       use TCP/IP device (error if multiple TCP/IP devices available)
 -s SERIAL                use device with given serial (overrides $ANDROID_SERIAL)
 -t ID                    use device with given transport id
 -H                       name of adb server host [default=localhost]
 -P                       port of adb server [default=5037]
 -L SOCKET                listen on given socket for adb server [default=tcp:localhost:5037]
 --one-device SERIAL|USB  only allowed with 'start-server' or 'server nodaemon', server will only connect to one USB device, specified by a serial number or USB device address.
 --exit-on-write-error    exit if stdout is closed

general commands:
 devices [-l]             list connected devices (-l for long output)
 help                     show this help message
 version                  show version num

networking:
 connect HOST[:PORT]      connect to a device via TCP/IP [default port=5555]
 disconnect [HOST[:PORT]]
     disconnect from given TCP/IP device [default port=5555], or all
 pair HOST[:PORT] [PAIRING CODE]
     pair with a device for secure TCP/IP communication
 forward --list           list all forward socket connections
 forward [--no-rebind] LOCAL REMOTE
     forward socket connection using:
       tcp:<port> (<local> may be "tcp:0" to pick any open port)
       localabstract:<unix domain socket name>
       localreserved:<unix domain socket name>
       localfilesystem:<unix domain socket name>
       jdwp:<process pid> (remote only)
       vsock:<CID>:<port> (remote only)
       acceptfd:<fd> (listen only)
 forward --remove LOCAL   remove specific forward socket connection
 forward --remove-all     remove all forward socket connections
 ppp TTY [PARAMETER...]   run PPP over USB
 reverse --list           list all reverse socket connections from device
 reverse [--no-rebind] REMOTE LOCAL
     reverse socket connection using:
       tcp:<port> (<remote> may be "tcp:0" to pick any open port)
       localabstract:<unix domain socket name>
       localreserved:<unix domain socket name>
       localfilesystem:<unix domain socket name>
 reverse --remove REMOTE  remove specific reverse socket connection
 reverse --remove-all     remove all reverse socket connections from device
 mdns check               check if mdns discovery is available
 mdns services            list all discovered services

file transfer:
 push [--sync] [-z ALGORITHM] [-Z] LOCAL... REMOTE
     copy local files/directories to device
     --sync: only push files that are newer on the host than the device
     -n: dry run: push files to device without storing to the filesystem
     -z: enable compression with a specified algorithm (any/none/brotli/lz4/zstd)
     -Z: disable compression
 pull [-a] [-z ALGORITHM] [-Z] REMOTE... LOCAL
     copy files/dirs from device
     -a: preserve file timestamp and mode
     -z: enable compression with a specified algorithm (any/none/brotli/lz4/zstd)
     -Z: disable compression
 sync [-l] [-z ALGORITHM] [-Z] [all|data|odm|oem|product|system|system_ext|vendor]
     sync a local build from $ANDROID_PRODUCT_OUT to the device (default all)
     -n: dry run: push files to device without storing to the filesystem
     -l: list files that would be copied, but don't copy them
     -z: enable compression with a specified algorithm (any/none/brotli/lz4/zstd)
     -Z: disable compression

shell:
 shell [-e ESCAPE] [-n] [-Tt] [-x] [COMMAND...]
     run remote shell command (interactive shell if no command given)
     -e: choose escape character, or "none"; default '~'
     -n: don't read from stdin
     -T: disable pty allocation
     -t: allocate a pty if on a tty (-tt: force pty allocation)
     -x: disable remote exit codes and stdout/stderr separation
 emu COMMAND              run emulator console command

app installation (see also `adb shell cmd package help`):
 install [-lrtsdg] [--instant] PACKAGE
     push a single package to the device and install it
 install-multiple [-lrtsdpg] [--instant] PACKAGE...
     push multiple APKs to the device for a single package and install them
 install-multi-package [-lrtsdpg] [--instant] PACKAGE...
     push one or more packages to the device and install them atomically
     -r: replace existing application
     -t: allow test packages
     -d: allow version code downgrade (debuggable packages only)
     -p: partial application install (install-multiple only)
     -g: grant all runtime permissions
     --abi ABI: override platform's default ABI
     --instant: cause the app to be installed as an ephemeral install app
     --no-streaming: always push APK to device and invoke Package Manager as separate steps
     --streaming: force streaming APK directly into Package Manager
     --fastdeploy: use fast deploy
     --no-fastdeploy: prevent use of fast deploy
     --force-agent: force update of deployment agent when using fast deploy
     --date-check-agent: update deployment agent when local version is newer and using fast deploy
     --version-check-agent: update deployment agent when local version has different version code and using fast deploy
     --local-agent: locate agent files from local source build (instead of SDK location)
     (See also `adb shell pm help` for more options.)
 uninstall [-k] PACKAGE
     remove this app package from the device
     '-k': keep the data and cache directories

debugging:
 bugreport [PATH]
     write bugreport to given PATH [default=bugreport.zip];
     if PATH is a directory, the bug report is saved in that directory.
     devices that don't support zipped bug reports output to stdout.
 jdwp                     list pids of processes hosting a JDWP transport
 logcat                   show device log (logcat --help for more)

security:
 disable-verity           disable dm-verity checking on userdebug builds
 enable-verity            re-enable dm-verity checking on userdebug builds
 keygen FILE
     generate adb public/private key; private key stored in FILE,

scripting:
 wait-for[-TRANSPORT]-STATE...
     wait for device to be in a given state
     STATE: device, recovery, rescue, sideload, bootloader, or disconnect
     TRANSPORT: usb, local, or any [default=any]
 get-state                print offline | bootloader | device
 get-serialno             print <serial-number>
 get-devpath              print <device-path>
 remount [-R]
      remount partitions read-write. if a reboot is required, -R will
      will automatically reboot the device.
 reboot [bootloader|recovery|sideload|sideload-auto-reboot]
     reboot the device; defaults to booting system image but
     supports bootloader and recovery too. sideload reboots
     into recovery and automatically starts sideload mode,
     sideload-auto-reboot is the same but reboots after sideloading.
 sideload OTAPACKAGE      sideload the given full OTA package
 root                     restart adbd with root permissions
 unroot                   restart adbd without root permissions
 usb                      restart adbd listening on USB
 tcpip PORT               restart adbd listening on TCP on PORT

internal debugging:
 start-server             ensure that there is a server running
 kill-server              kill the server if it is running
 reconnect                kick connection from host side to force reconnect
 reconnect device         kick connection from device side to force reconnect
 reconnect offline        reset offline/unauthorized devices to force reconnect

usb:
 attach                   attach a detached USB device
 detach                   detach from a USB device to allow use by other processes
environment variables:
 $ADB_TRACE
     comma-separated list of debug info to log:
     all,adb,sockets,packets,rwx,usb,sync,sysdeps,transport,jdwp
 $ADB_VENDOR_KEYS         colon-separated list of keys (files or directories)
 $ANDROID_SERIAL          serial number to connect to (see -s)
 $ANDROID_LOG_TAGS        tags to be used by logcat (see logcat --help)
 $ADB_LOCAL_TRANSPORT_MAX_PORT max emulator scan port (default 5585, 16 emus)
 $ADB_MDNS_AUTO_CONNECT   comma-separated list of mdns services to allow auto-connect (default adb-tls-connect)
```