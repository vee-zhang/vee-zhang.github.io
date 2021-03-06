# 电量优化


<!--more-->

## 电量优化方法

* 减少不必要的操作，例如缓存数据，而不每次都请求数据
* 延迟操作，等设备满电或充电时再执行。
* 合并操作，批处理，减少设备处于活动状态的时间。

## Doze打盹模式

Android6.0推出。

**针对整个系统**。

如果设备未充电，屏幕熄灭，让设备在一段时间后发生间歇性打盹-苏醒，打盹时阻塞所有任务，等苏醒时一并处理，且打盹时间会越来越长。

影响范围：

* 暂停网络
* 忽略PowerManager. WakeLock唤醒；
* 延迟AlarmManager.
* 不执行WIFI扫描；
* 不执行账号同步
* 不执行JobScheduler。

> 如果需要与网络建立持久性链接，应可能使用Firebase云消息传递FCM（以前叫GCM）。

### Doze机制下的闹钟处理

* `setAndAlarmWhileIdle()`一次性闹钟；
* `setExactAndAllowWhileIdle()`高精度一次性闹钟；
* `setAlarmClock`设置的闹钟可以正常触发，出发前系统会退出低电耗模式。

### Doze测试

#### 启动Doze

```shell
# 启动Doze
adb shell dumpsys deviceidle enable

# 强制启动Doze并关闭屏幕
adb shell dumpsys deviceidle force-idle
```

> 敲完命令后，必须要关闭屏幕才能真正进入Doze模式。

#### 退出Doze

```shell
# 关闭Doze
adb shell dumpsys deviceidle disable

# 退出Doze
adb shell dumpsys deviceidle unforce
```

#### 重置设备

```shell
# 重置电池设置
adb shell dumpsys battery reset
```

#### 查看Doze白名单

```shell
adb shell dumpsys deviceidle whitelist
```

### StandBy待机模式

**针对特定应用**。

应用待机模式会延迟用户近期未与之交互的应用的后台网络活动。

当用户很长时间没有与应用交互，并且，应用没有一下表现，则系统会使其进入空闲状态：

* 用户明确的启动应用；
* 有前台（活动或服务）进程在运行；
* 应用生成了通知。
  
当插入电源时，系统会从待机状态释放应用，如果设备长时间处于闲置状态，系统会允许限制的应用访问网络，频率大概一天一次。

### 电池优化白名单

在低电耗模式和应用待机模式期间，列入白名单的应用**可以使用网络并保留部分唤醒锁定**。不过，列入白名单的应用仍会受到其他限制，就像其他应用一样。例如，列入白名单的应用的作业和同步会延迟（在6.0及以下的设备上），并且其常规 AlarmManager 闹钟不会触发。

简言之，just agree but not easy。**仅仅同意使用网络和cpu唤醒，但是仍旧有限制，不保证你用的舒服**。

#### 检查是否在白名单内

```java
 PowerManager.isIgnoringBatteryOptimizations()
```

#### 跳转系统电池优化页面

```java
startActivity(new Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS));
```

#### 申请加入白名单

权限： `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` 。

该权限可以触发一个系统对话框，让用户直接将该应用添加到白名单，无需跳转到设置页面：

```java
<!-- 必须声明权限 -->
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />

Intent intent = new Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS);
intent.setData(Uri.parse("package:"+getPackageName()));
startActivity(intent);
```

#### StandBy模式测试

##### 断开充电

```shell
adb shell dumpsys battery unplug
```

##### 启停Standby

```shell
adb shell am set-inactive <包名> true/false
```

##### 查看是否处于Standby

```shell
adb shell am get-inactive <包名>
```

##### 重置电池设置

```shell 
adb shell dumpsys battery reset

```

### 监控电池状态

通过系统广播`Intent.ACTION_BATTERY_CHANGED`，可以监控到电池是否在充电，是否充满，用USB还是用充电器：

```java
IntentFilter ifilter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
Intent batteryStatus = registerReceiver(null, ifilter);
// 是否正在充电
int status = batteryStatus.getIntExtra(BatteryManager.EXTRA_STATUS, -1);
boolean isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING ||
status == BatteryManager.BATTERY_STATUS_FULL;
// 什么方式充电？
int chargePlug = batteryStatus.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
//usb
boolean usbCharge = chargePlug == BatteryManager.BATTERY_PLUGGED_USB;
//充电器
boolean acCharge = chargePlug == BatteryManager.BATTERY_PLUGGED_AC;
Log.e(TAG, "isCharging: " + isCharging + " usbCharge: " + usbCharge + " acCharge:" + acCharge);
```

系统广播 `Intent. ACTION_POWER_CONNECTED` 和 `Intent. ACTION_POWER_DISCONNECTED` 可以监控到电池状态的变化：

```java
/注册广播
IntentFilter ifilter = new IntentFilter();
//充电状态
ifilter.addAction(Intent.ACTION_POWER_CONNECTED);
ifilter.addAction(Intent.ACTION_POWER_DISCONNECTED);
//电量显著变化
ifilter.addAction(Intent.ACTION_BATTERY_LOW); //电量不足
ifilter.addAction(Intent.ACTION_BATTERY_OKAY); //电量从低变回高
powerConnectionReceiver = new PowerConnectionReceiver();
registerReceiver(powerConnectionReceiver, ifilter);
public class PowerConnectionReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent.getAction().equals(Intent.ACTION_POWER_CONNECTED)) {
          Toast.makeText(context, "充电状态：CONNECTED", Toast.LENGTH_SHORT).show();
        } else if (intent.getAction().equals(Intent.ACTION_POWER_DISCONNECTED)) {
          Toast.makeText(context, "充电状态：DISCONNECTED", Toast.LENGTH_SHORT).show();
        } else if (intent.getAction().equals(Intent.ACTION_BATTERY_LOW)) {
          Toast.makeText(context, "电量过低", Toast.LENGTH_SHORT).show();
        } else if (intent.getAction().equals(Intent.ACTION_BATTERY_OKAY)) {
          Toast.makeText(context, "电量从低变回高", Toast.LENGTH_SHORT).show();
        }
    }
}

```

### WorkManager

WorkManager可以根据电池状态调度后台任务。

### Battery Historian工具

Battery Historian是一个可以了解设备随时间的耗电情况的工具 。在系统级别，该工具以 HTML 的形式可视化来
自系统日志的电源相关事件。在具体应用级别，该工具可提供各种数据，帮助您识别耗电的应用行为：

* 过于频繁的触发唤醒提醒（至少每10秒一次）；
* 持续保留GPS锁定；
* 至少每30秒调度一次作业；
* 至少每30秒调度一次同步；
* 使用手机无线装置的频率高于预期。

依赖GO、Python和Git，拉倒吧~

### Profile工具
