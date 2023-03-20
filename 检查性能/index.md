# 检查性能

由于负责系统稳定性和性能监控的同事离职了，所以领导希望我来与剩下的同事一起负责这个事，那没法了，学吧～
<!--more-->
## 捕获跟踪记录的方式

方式：

- Android Studio 上面有个profile可视化工具，可以在AS中启动，也可以独立使用。
- 系统跟踪使用程序（System Tracing）
- Perfetto（Android10及以上）
- Systrace 命令行工具

对比一下：

1. profile工具依赖AS，图形化，非常方便，但是无法远程操作；
2. System Tracing在国内很多系统上都阉割掉了，比如我的华为mate20x；
3. Perfetto 是Android10及以上才加入，老系统不支持（所幸我们的系统还是支持的），同事强烈推荐，但是因为适用性较少，放在后面去学吧；
4. Systrace 命令行工具，适用性最广，面试经常考，所以优先掌握。

## Systrace

### 条件

- Android SDK
- Python 最好是Python2.7，其他版本我没有试
- android-sdk/platform-tools/添加到PATH环境变量，将android-sdk/platform-tools/systrace目录也添加进PATH。

使用如下命令测试：

```
$ python systrace.py --list-categories
```

这里我就出错了：`python: can't open file 'systrace.py': [Errno 2] No such file or directory`。

到这里我先搜索百度，查到systrace.py是位于android-sdk/platform-tools，但是进入到目录之后发现文件并不存在，于时又查到google的platform-tools包中，在33.0.0以后已经删掉了这个工具，所以我们需要额外下载33.0.0或更早的platform-tools才行，这里给出下载链接
- [linux](https://links.jianshu.com/go?to=https%3A%2F%2Fdl.google.com%2Fandroid%2Frepository%2Fplatform-tools_r33.0.0-linux.zip)
- [windows](https://links.jianshu.com/go?to=https%3A%2F%2Fdl.google.com%2Fandroid%2Frepository%2Fplatform-tools_r33.0.0-windows.zip)
- [darwin](https://links.jianshu.com/go?to=https%3A%2F%2Fdl.google.com%2Fandroid%2Frepository%2Fplatform-tools_r33.0.0-darwin.zip)

>感谢[RedB](http://events.jianshu.io/p/626eaebaa6a8)的博客。

进入目录`android-sdk/platform-tools/trace/`下，发现是`systrace.py`文件是存在的，然后提权`sudo chmod 777 systrace.py`后仍然报以前的错误。

后来尝试如下命令成功：

```
$ systrace.py --list-categories
```

### 语法

```
python systrace.py [options] [categories]
```

#### 选项

|全局选项|说明|
|---|---|
|-h|显示帮助信息|
|-l|列出您的已连接设备科用的跟踪类型

|命令和选项|说明|
|---|---|
|-o `file`|将HTML跟踪报告写入指定的`file`，如果没有指定此选项，则保存在当前目录的trace.html|
|-t `N`|跟踪设备活动`N`秒|
|-b `N`|使用`N`千字节的缓冲区大小|
|-k `functions`|跟踪逗号分隔列表中特定的内核函数活动|
|-a `app-name`| 启动对应用的跟踪，指定为包含进程名称的逗号分隔列表，必须包含Trace类中的插桩调用|
|-e `device-serial`|指定特定的设备|
|`categories`|包含指定的系统进程的跟踪信息|
