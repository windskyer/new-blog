---
title: IPVS 简易入门
categories: Kubernetes
sage: false
date: 2021-06-12 11:23:17
tags: ipvs
---

# IPVS简介

我们接触比较多的是应用层负载均衡，比如haproxy、nginx、F5等，这些负载均衡工作在用户态，因此会有对应的进程和监听socket，一般能同时支持4层负载和7层负载，使用起来也比较方便。

LVS是国内章文嵩博士开发并贡献给社区的（章文嵩博士和他背后的负载均衡帝国 )，主要由ipvs和ipvsadm组成，ipvs是工作在内核态的4层负载均衡，和iptables一样都是基于内核底层netfilter实现，netfilter主要通过各个链的钩子实现包处理和转发。ipvsadm和ipvs的关系，就好比netfilter和iptables的关系，它运行在用户态，提供简单的CLI接口进行ipvs配置。

<!-- more -->

由于ipvs工作在内核态，直接基于内核处理包转发，所以最大的特点就是性能非常好。又由于它工作在4层，因此不会处理应用层数据，经常有人问ipvs能不能做SSL证书卸载、或者修改HTTP头部数据，显然这些都不可能做的。

我们知道应用层负载均衡大多数都是基于反向代理实现负载的，工作在应用层，当用户的包到达负载均衡监听器listening后，基于一定的算法从后端服务列表中选择其中一个后端服务进行转发。当然中间可能还会有一些额外操作，最常见的如SSL证书卸载。

而ipvs工作在内核态，只处理四层协议，因此只能基于路由或者NAT进行数据转发，可以把ipvs当作一个特殊的路由器网关，这个网关可以根据一定的算法自动选择下一跳，或者把ipvs当作一个多重DNAT，按照一定的算法把ip包的目标地址DNAT到其中真实服务的目标IP。针对如上两种情况分别对应ipvs的两种模式–网关模式和NAT模式，另外ipip模式则是对网关模式的扩展，本文下面会针对这几种模式的实现原理进行详细介绍。

# IPVS用法

ipvsadm命令行用法和iptables命令行用法非常相似，毕竟是兄弟，比如-L列举，-A添加，-D删除。

```sh
ipvsadm -A -t 10.254.0.1:32016 -s rr
```

但是其实ipvsadm相对iptables命令简直太简单了，因为没有像iptables那样存在各种table，table嵌套各种链，链里串着一堆规则，ipvsadm就只有两个核心实体，分别为service和server，service就是一个负载均衡实例，而server就是后端member,ipvs术语中叫做real server，简称RS。

如下命令创建一个service实例172.17.0.1:32016，-t指定监听的为TCP端口，-s指定算法为轮询算法rr(Round Robin)，ipvs支持简单轮询(rr)、加权轮询(wrr)、最少连接(lc)、源地址或者目标地址散列(sh、dh)等10种调度算法。

```sh
ipvsadm -A -t 172.17.0.1:32016 -s rr
```

然后把10.244.1.2:8080、10.244.1.3:8080、10.244.3.2:8080添加到service后端member中。

```sh
ipvsadm -a -t 172.17.0.1:32016 -r 10.244.1.2:8080 -m -w 1
ipvsadm -a -t 172.17.0.1:32016 -r 10.244.1.3:8080 -m -w 1
ipvsadm -a -t 172.17.0.1:32016 -r 10.244.3.2:8080 -m -w 1
```

其中-t指定service实例，-r指定server地址，-w指定权值，-m即前面说的转发模式，其中-m表示为masquerading，即NAT模式，-g为gatewaying，即直连路由模式，-i为ipip,即IPIP隧道模式。
与iptables-save、iptables-restore对应的工具ipvs也有ipvsadm-save、ipvsadm-restore。

# 工作模式

## NAT(network access translation)模式

NAT模式由字面意思理解就是通过NAT实现的，但究竟是如何NAT转发的，我们通过实验环境验证下。

现环境中LB节点IP为10.0.0.161，两个RS节点如下:

>- 10.0.0.42:80
>- 10.0.0.78:80

为了模拟LB节点IP和RS不在同一个网络的情况，在LB节点中添加一个虚拟IP地址:

```sh
ip addr add 10.254.0.1/24 dev eth0
```

创建负载均衡Service并把RS添加到Service中:

```sh
ipvsadm -A -t 10.254.0.1:8080 -s rr
ipvsadm -a -t 10.254.0.1:8080 -r 10.0.0.42:80 -m
ipvsadm -a -t 10.254.0.1:8080 -r 10.0.0.78:80 -m
```

这里需要注意的是，和应用层负载均衡如haproxy、nginx不一样的是，haproxy、nginx进程是运行在用户态，因此会创建socket，本地会监听端口，而ipvs的负载是直接运行在内核态的，因此不会出现监听端口:

```sh
root@ip-192-168-193-197:/var/log# netstat -lnpt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      674/systemd-resolve
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      950/sshd
tcp6       0      0 :::22                   :::*                    LISTEN      950/sshd
```

可见并没有监听10.222.0.1:8080 Socket。

Client节点IP为10.0.0.8，为了和LB节点的虚拟IP 10.254.0.1通，我们手动添加静态路由如下:
```sh
ip r add 10.254.0.1 via 10.0.0.161 dev eth0
```
此时Client节点能够ping通LB节点VIP:

```sh
[root@vm10-0-0-8 ~]# ping -c 2 -w 2 10.254.0.1 
PING 10.254.0.1 (10.254.0.1) 56(84) bytes of data.
64 bytes from 10.254.0.1: icmp_seq=1 ttl=64 time=0.310 ms
64 bytes from 10.254.0.1: icmp_seq=2 ttl=64 time=0.184 ms

--- 10.254.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.184/0.247/0.310/0.063 ms
```

可见Client节点到VIP的链路没有问题，那是否能够访问我们的Service呢？

我们验证下:

```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL  10.254.0.1:8080
curl: (28) Connection timed out after 2001 milliseconds
```

非常意外的结果是并不通。

在RS节点抓包如下:

```sh
[root@vm10-0-0-42 ~]# tcpdump  -t -ne -i eth0 -nn port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes


fa:16:3e:30:64:a2 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 74: 10.0.0.8.16352 > 10.0.0.42.80: Flags [S], seq 2572069217, win 29200, options [mss 1460,sackOK,TS val 559733 ecr 0,nop,wscale 7], length 0
fa:16:3e:12:8c:cd > fa:16:3e:44:bf:83, ethertype IPv4 (0x0800), length 74: 10.0.0.42.80 > 10.0.0.8.16352: Flags [S.], seq 2421605018, ack 2572069218, win 28960, options [mss 1460,sackOK,TS val 2392752 ecr 559733,nop,wscale 7], length 0
fa:16:3e:44:bf:83 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 54: 10.0.0.8.16352 > 10.0.0.42.80: Flags [R], seq 2572069218, win 0, length 0
fa:16:3e:30:64:a2 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 74: 10.0.0.8.16352 > 10.0.0.42.80: Flags [S], seq 2572069217, win 29200, options [mss 1460,sackOK,TS val 560736 ecr 0,nop,wscale 7], length 0
fa:16:3e:12:8c:cd > fa:16:3e:44:bf:83, ethertype IPv4 (0x0800), length 74: 10.0.0.42.80 > 10.0.0.8.16352: Flags [S.], seq 2437261525, ack 2572069218, win 28960, options [mss 1460,sackOK,TS val 2393754 ecr 560736,nop,wscale 7], length 0
fa:16:3e:44:bf:83 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 54: 10.0.0.8.16352 > 10.0.0.42.80: Flags [R], seq 2572069218, win 0, length 0
```

我们发现数据包的源IP为Client IP，目标IP为RS IP，换句话说，LB节点IPVS只做了DNAT，把目标IP改成RS IP了，而没有修改源IP。此时虽然RS和Client在同一个子网，链路连通性没有问题，但是由于Client节点发出去的包的目标IP和收到的包源IP不一致，因此会被直接丢弃，相当于给张三发信，李四回的信，显然不受信任。

既然IPVS没有给我们做SNAT，那自然想到的是我们手动做SNAT，在LB节点添加如下iptables规则：

```sh
iptables -t nat -A POSTROUTING -m ipvs  --vaddr 10.254.0.1 --vport 8080 -j LOG --log-prefix '[int32bit ipvs]'
iptables -t nat -A POSTROUTING -m ipvs  --vaddr 10.254.0.1 --vport 8080 -j MASQUERADE
```

再次检查Service是否可以访问:

```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL  10.254.0.1:8080
curl: (28) Connection timed out after 2001 milliseconds
```

服务依然不通。并且在LB节点的iptables日志为空:

```sh
cat /var/log/messages | grep 'int32bit ipvs'
```

也就是说，ipvs的包根本不会经过iptables nat表POSTROUTING链？

那mangle表呢？我们打开LOG查看下:

```sh
iptables -t mangle -A POSTROUTING -m ipvs --vaddr 10.254.0.1 --vport 8080 -j LOG --log-prefix "[int32bit ipvs]"
```

此时查看日志如下：

```sh
[root@vm10-0-0-161 ~]# cat /var/log/messages | grep 'int32bit ipvs'
Nov 22 16:18:02 vm10-0-0-161 kernel: [int32bit ipvs]IN= OUT=eth0 SRC=10.0.0.8 DST=10.0.0.42 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=44784 DF PROTO=TCP SPT=16648 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 
Nov 22 16:18:03 vm10-0-0-161 kernel: [int32bit ipvs]IN= OUT=eth0 SRC=10.0.0.8 DST=10.0.0.42 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=44785 DF PROTO=TCP SPT=16648 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 
Nov 22 16:18:05 vm10-0-0-161 kernel: [int32bit ipvs]IN= OUT=eth0 SRC=10.0.0.8 DST=10.0.0.78 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=28036 DF PROTO=TCP SPT=16650 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 
Nov 22 16:18:06 vm10-0-0-161 kernel: [int32bit ipvs]IN= OUT=eth0 SRC=10.0.0.8 DST=10.0.0.78 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=28037 DF PROTO=TCP SPT=16650 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0 
```

我们发现在mangle表中可以看到DNAT后的包。

只是可惜mangle表的POSTROUTING并不支持NAT功能:
```sh
iptables -t mangle  -A POSTROUTING -m ipvs  --vaddr 10.254.0.1 --vport 8080 -j MASQUERADE
iptables: Invalid argument. Run `dmesg' for more information.
[ 6989.347012] x_tables: ip_tables: MASQUERADE target: only valid in nat table, not mangle
```

对比Kubernetes配置发现需要设置如下系统参数:

```sh
sysctl net.ipv4.vs.conntrack=1
```

再次验证：

```sh
 curl -m 2 --retry 1 -sSL  10.254.0.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

终于通了，查看RS抓包：

```sh
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes





fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 74: 10.0.0.161.16676 > 10.0.0.78.80: Flags [S], seq 1625080630, win 29200, options [mss 1460,sackOK,TS val 5044280 ecr 0,nop,wscale 7], length 0
fa:16:3e:37:86:c3 > fa:16:3e:30:64:a2, ethertype IPv4 (0x0800), length 74: 10.0.0.78.80 > 10.0.0.161.16676: Flags [S.], seq 985612622, ack 1625080631, win 28960, options [mss 1460,sackOK,TS val 6877793 ecr 5044280,nop,wscale 7], length 0
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 66: 10.0.0.161.16676 > 10.0.0.78.80: Flags [.], ack 1, win 229, options [nop,nop,TS val 5044281 ecr 6877793], length 0
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 145: 10.0.0.161.16676 > 10.0.0.78.80: Flags [P.], seq 1:80, ack 1, win 229, options [nop,nop,TS val 5044281 ecr 6877793], length 79: HTTP: GET / HTTP/1.1
fa:16:3e:37:86:c3 > fa:16:3e:30:64:a2, ethertype IPv4 (0x0800), length 66: 10.0.0.78.80 > 10.0.0.161.16676: Flags [.], ack 80, win 227, options [nop,nop,TS val 6877794 ecr 5044281], length 0
fa:16:3e:37:86:c3 > fa:16:3e:30:64:a2, ethertype IPv4 (0x0800), length 304: 10.0.0.78.80 > 10.0.0.161.16676: Flags [P.], seq 1:239, ack 80, win 227, options [nop,nop,TS val 6877794 ecr 5044281], length 238: HTTP: HTTP/1.1 200 OK
fa:16:3e:37:86:c3 > fa:16:3e:30:64:a2, ethertype IPv4 (0x0800), length 678: 10.0.0.78.80 > 10.0.0.161.16676: Flags [P.], seq 239:851, ack 80, win 227, options [nop,nop,TS val 6877794 ecr 5044281], length 612: HTTP
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 66: 10.0.0.161.16676 > 10.0.0.78.80: Flags [.], ack 239, win 237, options [nop,nop,TS val 5044281 ecr 6877794], length 0
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 66: 10.0.0.161.16676 > 10.0.0.78.80: Flags [.], ack 851, win 247, options [nop,nop,TS val 5044281 ecr 6877794], length 0
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 66: 10.0.0.161.16676 > 10.0.0.78.80: Flags [F.], seq 80, ack 851, win 247, options [nop,nop,TS val 5044282 ecr 6877794], length 0
fa:16:3e:37:86:c3 > fa:16:3e:30:64:a2, ethertype IPv4 (0x0800), length 66: 10.0.0.78.80 > 10.0.0.161.16676: Flags [F.], seq 851, ack 81, win 227, options [nop,nop,TS val 6877794 ecr 5044282], length 0
fa:16:3e:30:64:a2 > fa:16:3e:37:86:c3, ethertype IPv4 (0x0800), length 66: 10.0.0.161.16676 > 10.0.0.78.80: Flags [.], ack 852, win 247, options [nop,nop,TS val 5044282 ecr 6877794], length 0
```

如期望，修改了源IP为LB IP。

原来需要配置net.ipv4.vs.conntrack=1参数，这个问题折腾了一个晚上，不得不说目前ipvs的文档都太老了。

前面是通过手动iptables实现SNAT的，性能可能会有损耗，于是如下开源项目通过修改lvs直接做SNAT:

1. 小米运维部在LVS的FULLNAT基础上，增加了SNAT网关功能，参考xiaomi-sa/dsnat
2. lvs-snat

除了SNAT的办法，是否还有其他办法呢？想想我们最初的问题，Client节点发出去的包的目标IP和收到的包源IP不一致导致包被丢弃，那解决问题的办法就是把包重新引到LB节点上，只需要在所有的RS节点增加如下路由即可:
```sh
 ip r add 10.0.0.8 via 10.0.0.161 dev eth0
```
此时我们再次检查我们的Service是否可连接:

```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL  10.254.0.1:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

结果没有问题。

不过我们是通过手动添加Client IP到所有RS的明细路由实现的，如果Client不固定，这种方案仍然不太可行，所以通常做法是干脆把所有RS默认路由指向LB节点，即把LB节点当作所有RS的默认网关。

由此可知，用户通过LB地址访问服务，LB节点IPVS会把用户的目标IP由LB IP改为RS IP，源IP不变，包不经过iptables的OUTPUT直接到达POSTROUTING转发出去，包回来的时候也必须先到LB节点，LB节点把目标IP再改成用户的源IP，最后转发给用户。

显然这种模式来回都需要经过LB节点，因此又称为双臂模式。

## 网关(Gatewaying)模式

网关模式（Gatewaying）又称为直连路由模式（Direct Routing）、透传模式，所谓透传即LB节点不会修改数据包的源IP、端口以及目标IP、端口，LB节点做的仅仅是路由转发出去，可以把LB节点看作一个特殊的路由器网关，而RS节点则是网关的下一跳，这就相当于对于同一个目标地址，会有多个下一跳，这个路由器网关的特殊之处在于能够根据一定的算法选择其中一个RS作为下一跳，达到负载均衡和冗余的效果。

既然是通过直连路由的方式转发，那显然LB节点必须与所有的RS节点在同一个子网，不能跨子网，否则路由不可达。换句话说，这种模式只支持内部负载均衡(Internal LoadBalancer)。

另外如前面所述，LB节点不会修改源端口和目标端口，因此这种模式也无法支持端口映射，换句话说LB节点监听的端口和所有RS节点监听的端口必须一致。

现在假设有LB节点IP为 10.0.0.161，有两个RS节点如下:
>- 10.0.0.78:80
>- 10.0.0.42:80

创建负载均衡Service并把RS添加到Service中:

```sh
ipvsadm -A -t 10.0.0.161:80 -s rr
ipvsadm -a -t 10.0.0.161:80 -r 10.0.0.78:80 -g
ipvsadm -a -t 10.0.0.161:80 -r 10.0.0.42:80 -g
```
注意到我们的Service监听的端口80和RS的端口是一样的，并且通过-g参数指定为直连路由模式(网关模式)。

Client节点IP为10.0.0.8，我们验证Service是否可连接：
```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL  10.0.0.161:80
curl: (28) Connection timed out after 2001 milliseconds
```

我们发现并不通，在其中一个RS节点 10.0.0.78 上抓包:
```sh
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

fa:16:3e:30:64:a2 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 74: 10.0.0.8.43130 > 10.0.0.161.80: Flags [S], seq 2973200786, win 29200, options [mss 1460,sackOK,TS val 7531783 ecr 0,nop,wscale 7], length 0
fa:16:3e:30:64:a2 > fa:16:3e:12:8c:cd, ethertype IPv4 (0x0800), length 74: 10.0.0.8.43130 > 10.0.0.161.80: Flags [S], seq 2973200786, win 29200, options [mss 1460,sackOK,TS val 7532784 ecr 0,nop,wscale 7], length 0
```

正如前面所说，LB是通过路由转发的，根据路由的原理，源MAC地址修改为LB的MAC地址，而目标MAC地址修改为RS MAC地址，相当于RS是LB的下一跳。

并且源IP和目标IP都不会修改。问题就来了，我们Client期望访问的是RS，但RS收到的目标IP却是LB的IP，发现这个目标IP并不是自己的IP，因此不会通过INPUT链转发到用户空间，这时要不直接丢弃这个包，要不根据路由再次转发到其他地方，总之两种情况都不是我们期望的结果。

那怎么办呢？为了让RS接收这个包，必须得让RS有这个目标IP才行。于是不妨在lo上添加个虚拟IP，IP地址伪装成LB IP 10.0.0.161:
```sh
ifconfig lo:0 10.0.0.161/32
```

问题又来了，这就相当于有两个相同的IP，IP重复了怎么办？办法是隐藏这个虚拟网卡，不让它回复ARP，其他主机的neigh也就不可能知道有这么个网卡的存在了，参考[Using arp announce/arp ignore to disable ARP。](http://kb.linuxvirtualserver.org/wiki/Using_arp_announce/arp_ignore_to_disable_ARP)

```sh
sysctl net.ipv4.conf.lo.arp_ignore=1
sysctl net.ipv4.conf.lo.arp_announce=2
```

此时再次从客户端curl:

```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL 10.0.0.161:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
终于通了。

我们从前面的抓包中知道，源IP为Client IP 10.0.0.8，因此直接回包给Client即可，不可能也不需要再回到LB节点了，即A->B, B->C, C->A，流量方向是三角形状的，因此这种模式又称为三角模式。

我们从原理中不难得出如下结论：

>1. Client、LB以及所有的RS必须在同一个子网。
>2. LB节点直接通过路由转发，因此性能非常高。
>3. 不能做端口映射。

## ipip隧道模式

前面介绍了网关直连路由模式，要求所有的节点在同一个子网，而ipip隧道模式则主要解决这种限制，LB节点IP和RS可以不在同一个子网，此时需要通过ipip隧道进行传输。

现在假设有LB节点IP为 10.0.10.13/24:

有两个RS节点如下:

>- 10.0.0.42:80
>- 10.0.0.78:80

如上两个RS节点子网掩码均为255.255.255.0，即24位子网，显然和VIP 10.0.10.13不在同一个子网。

创建负载均衡Service并把RS添加到Service中:

```sh
ipvsadm -A -t 10.0.10.13:80 -s rr
ipvsadm -a -t 10.0.10.13:80 -r 10.0.0.42:80 -i
ipvsadm -a -t 10.0.10.13:80 -r 10.0.0.78:80 -i
```
注意到我们的Service监听的端口80和RS的端口是一样的，并且通过-i参数指定为ipip隧道模式。

在所有的RS节点上加载ipip模块以及添加VIP(和直连路由类型）:

```sh
modprobe ipip
ifconfig tunl0 10.0.10.13/32
sysctl net.ipv4.conf.tunl0.arp_ignore=1
sysctl net.ipv4.conf.tunl0.arp_announce=2
```

Client节点IP为10.0.0.8/24，我们验证Service是否可连接：
```sh
[root@vm10-0-0-8 ~]# curl -m 2 --retry 1 -sSL 10.0.10.13:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Service可访问，我们在RS节点上抓包如下：



我们发现和直连路由一样，源IP和目标IP没有修改。

所以IPIP模式和网关(Gatewaying)模式原理基本一样，唯一不同的是网关(Gatewaying)模式要求所有的RS节点和LB节点在同一个子网，而IPIP模式则可以支持跨子网的情况，为了解决跨子网通信问题，使用了ipip隧道进行数据传输。

# 总结

ipvs是一个内核态的四层负载均衡，支持NAT、Gateway以及IPIP隧道模式，Gateway模式性能最好，但LB和RS不能跨子网，IPIP性能次之，通过ipip隧道解决跨网段传输问题，因此能够支持跨子网。而NAT模式没有限制，这也是唯一一种支持端口映射的模式。

我们不难猜想，由于Kubernetes Service需要使用端口映射功能，因此kube-proxy必然只能使用ipvs的NAT模式。
