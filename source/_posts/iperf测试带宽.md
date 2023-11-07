---
title: iperf测试带宽
categories: Kubernetes
sage: false
date: 2021-06-22 11:47:14
tags: iperf
---

# iperf测试带宽

iperf是开源的跨平台网络带宽测试工具，支持windows、linux、macos、bsd等众多系统。可以测试带宽吞吐量、延迟、丢包等；支持使用TCP和UDP测试，结果比较准。由于是C/S架构，使用时需要在测试宽带的两端分别运行一个装有iperf的电脑，我是测试连接两个机房的专线，直接拿机房里的Linux服务器测试。
iperf有两个版本，一个是iperf2和iperf3,建议使用iperf3,iperf2比较老，对10G网卡测试速度上不来。两个版本用法基本上是一样的，我这里是测试百兆的带宽，以ipser2来做演示
<!-- more -->

## 安装

```sh
yum install iperf iperf3   #需要安装epel源
```
## 测试

无论是tcp还是udp方式测试，都要一端运行服务器模式，另一端运行客户端模式,另外如果开了iptables,要打开tcp 5001端口,当然也可以指定端口。

### tcp方式

服务器端：
```sh
iperf -s
```

客户端：
```sh
iperf -c SERVERIP -t 60 -i 1
```

在实际测试中（100M MPLS专线），使用tcp方式测试带宽经常跑不满，联系供应商后得到回复说是因为“TCP窗口大小”导致的。iperf也有一个“-w”参数可以调整TCP窗口大小值，但是无论改TCP窗口大小设置成多少，就是跑不满带宽。我猜测是因为单线程的tcp的传输效率有限,所以无论tcp窗口设置成多大，也不可能跑满全部带宽。在内网环境使用iperf测试时，网络条件比较好，多线程和单线程测试结果基本上没有区别。但是在测试专线时，合适的线程数是可以跑满带宽的，设置太大反而速度会下降。

```sh
iperf -c SERVERIP -t 60 -i 1 -P 5 
```

### udp方式
服务器端：
```sh
iperf -u -s
```

客户端：
```sh
iperf -c SERVERIP -t 60 -i 1 -b 100M
```
注意：udp是无连接不可靠的，只管发包不确保接收，发包速度非常快。如果-b指定的带宽值过大（例如100M专线，这里设置成200M），会将整个宽带占满，起到了dos的效果，这时如果你是用ssh连接到两端的服务器就悲剧了（tcp方式就不会），别问我是怎么知道的 TT~

参数说明：
linux下iperf 2.0.5 版本
适用客户端/服务器：
```sh
-f --格式[k|m|K|M] 分别表示以Kbits、Mbits、KBytes、MBytes显示报告，默认是Mbits
-i 以秒为单位统计带宽值 
-l 读写缓冲区大小，默认是8KB
-m 显示最大的TCP数据段大小 (MTU - TCP/IP header)
-o 将报告和错误信息输出到文件
-p 指定服务器和客户端连接的端口
-u 使用udp协议
-w 指定TCP窗口大小，默认是8KB
-B 绑定一个主机地址或接口（当主机有多个地址或接口时使用该参数）
-C 兼容旧版本（当server端和client端版本不一样时使用）
-M 设定TCP数据包的最大mtu值
-N 设定TCP不延时
-V 传输ipv6数据包
```

适用服务器端：

```sh
-s 以服务器模式启动
-U 单线程UDP模式下运行
-D 以守护进程模式运行
```

适用客户端：

```sh
-b 指定客户端通过UDP协议发送信息的带宽，默认值为1Mbit/s
-c 指定服务器地址　　
-d 同时进行双向传输测试
-n 指定传输的字节数
-r 单独进行双向传输测试
-t 指定Iperf测试时间，默认10秒
-F 指定需要传输的文件
-I 从标准输入（stdin）中读取要传输的数据 
-L 指定一个端口，服务器将利用这个端口与客户机连接
-P 客户端到服务器的连接数，默认值为1
-T 指定ttl值
```

参考文章
http://linux-guys.blogspot.com/2011/01/iperf.html
http://wenku.baidu.com/view/4cedc8728e9951e79b8927a7.html