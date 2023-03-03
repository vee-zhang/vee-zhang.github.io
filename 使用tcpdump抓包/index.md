# 使用tcpdump抓包

最近搞PKI，遇到问题经常不明所以，后来找同事帮忙抓包才定位到原因，瞬间感觉学习抓包的重要。
<!--more-->

## 为什么要使用tcpdump

据了解`tcpdump`工具在unix中内置了，所以一般的什么ubuntu、Android、mac系统中都内置了，不需要额外下载安装，尤其是在Android系统中使用非常方便。

本文主要参考[肝了三天，万字长文教你玩转 tcpdump，从此抓包不用愁](https://blog.51cto.com/u_15009285/2552501),一边学习，一遍整理成自己容易记忆的文字，感谢原作！

## 命令格式

eg：

```shell
tcpdump src host 192.168.10.100
```

![tcpdump参数图解](tcpdump.png)

1. option 可选参数;
2. proto 协议过滤器，根据协议进行过滤，关键词：upd,udp,icmp,ip,ip6,arp,rarp,ether,wlan,fddi,tr,decnet；
3. direction 定向过滤器，根据数据流向进行过滤，关键词：src,dst,可以通过逻辑运算符进行组合，如：src or dst；
4. type 类过滤器，关键词：host,net,port,portrange。

## 输出格式

```shell
21:26:49.013621 IP 172.20.20.1.15605 > 172.20.20.2.5920: Flags [P.], seq 49:97, ack 106048, win 4723, length 48
```

1. 时间：时：分：秒.毫秒
2. 网络协议：IP
3. 发送方ip.端口号
4. 箭头表示数据流向
5. 接收方ip.端口号
6. 冒号
7. 数据包内容，包括Flags标识符，seq号，ack号，win窗口，数据长度length。

### Flags 标识符

- `S`：SYN，开始链接
- `P`：PSH，推送数据
- `F`：FIN,结束连接
- `R`：RST,重置连接
- `.`：没有FLAG，由于除了SYN之外所有的数据包都有ACK，所以一般这个标志也可以表示ACK。

## 常规过滤规则

### IP和域名过滤

```shell
tcpdump host 192.168.10.100

#根据源IP过滤
tcpdump -i eth2 src 192.168.10.100

#根据目标IP过滤
tcpdump -i eth2 dst 192.168.10.200
```

### 网段过滤

```shell
tcpdump net 192.168.10.0/24

#源网段
tcpdump src net 192.168.10.11

#目标网段
tcpdump src net 192.168.10.11
```

### 端口过滤

```shell
tcpdump port 8088

# 源端口
tcpdump src port 8088

# 目标端口
tcpdump dst port 8088

# 多个端口
tcpdump port 80 or 8088

# 范围抓取
tcpdump portrange 8000-8080$ tcpdump src portrange 8000-8080$ tcpdump dst portrange 8000-8080
```

### 协议过滤

```
tcpdump tcp
```

可选值：

protocol 可选值：
- ip
- ip6
- arp
- rarp
- atalk
- aarp
- decnet
- sca
- lat
- mopdl
- moprc
- iso
- stp
- ipx
- netbeui

## 可选参数

### 不解析域名提升速度

- `-n`:不把ip转化成域名，直接先时ip，避免执行DNS lookup以提高速度；
- `-nn`:不把协议和端口号转化成名字，也能提高速度；
- `-N`：不打印出host的域名部分。

### 过滤结果输出到文件

`-w`参数后接一个以`.pcap`为后缀的文件名。

```
tcpdump icmp -w icmp.pcap
```

### 从文件读取包数据

```
tcpdump icmp -r icmp.pcap
```

### 控制输出

- `-v` ：产生详细的输出，比如包的TTL，id标识，数据包长度，以及IP包的一些选项。同事它还会打开一些附加的包的完整性检测，比如对IP或ICMP包头部的校验；
- `-vv`: 产生更加详细的输出；
- `-vvv`:产生最详细的输出。

### 控制时间

- `-t`:不输出时间
- `-tt`:输出时间戳
- `-ttt`:输出每两行打印的时间间隔（毫秒）
- `-tttt`:在每行打印的时间戳之前添加日期的打印

### 显示数据包头部

- `-x`:以16进制打印每个包头部数据，不包括数据链路层；
- `-xx`：同上，但包括数据链层；
- `-X`：以16进制和 ASCII码形式打印出每个包的数据(但不包括连接层的头部)，这在分析一些新协议的数据包很方便；
- `-XX`:以16进制和 ASCII码形式打印出每个包的数据(包括连接层的头部)，这在分析一些新协议的数据包很方便。

### 过滤指定网卡的数据包

- `-i`:指定要过滤的网卡接口，如果要查看所有网卡，可以`-i any`

### 过滤特定流向的数据包

```shell
// 入方向
tcpdump -Q in

// 出方向
tcpdump -Q out

// 都要
tcpdump -Q inout
```

### 其他常用参数

- `-A`：以ASCII码方式显示每一个数据包；
- `-l`：基于行输出；
- `-q`：简洁输出；
- `-c`：捕获count个包之后退出；
- `-s`：设置要截取的包文字节数，默认**96**字节，设为**0**则截取全部报文；
- `-C`：设置`-w`文件的大小，超出大小将新建另一个文件；
- `-F`：使用file文件作为过滤条件的表达式输入。

### 输出控制参数

- `-D`：显示所有可用网络接口的列表；
- `-e`：每行的打印；
- `-E`：解密IPSEC数据；
- `-L`：列出指定网络接口所支持的数据链路层的类型后退出；
- `-Z`：后接用户名，在抓包时会收到权限的限制，如果以root启动，泽tcpdump会用有超级用户权限；
- `-d`：打印出易读的包匹配码；
- `-dd`：以C语言的形式打印出包匹配码；
- `-ddd`：以十进制数的形式打印出包匹配码。

## 逻辑过滤

tcpdump可以通过逻辑运算符配置出更加灵活的过滤规则：

- and
- or
- not
- ==
- !=
- if
- proc：进程名
- pid：进程号
- svc：service class
- dir：表示方向，in 和 out
- eproc：表示effective process name
- epid：表示effective process ID

eg:

比如我现在要过滤来自进程名为 nc 发出的流经 en0 网卡的数据包，或者不流经 en0 的入方向数据包，可以这样子写:

```shell
tcpdump "( if=en0 and proc =nc ) || (if != en0 and dir=in)"
```

## 特殊过滤规则

### 过滤mac地址

```
tcpdump ether host [ehost]$ tcpdump ether dst    [ehost]$ tcpdump ether src    [ehost]
```

### 过滤网关

```
tcpdump gateway [host]
```

### 过滤广播/多播数据包

```
tcpdump ether broadcast$ tcpdump ether multicast$ tcpdump ip broadcast$ tcpdump ip multicast$ tcpdump ip6 multicast
```
