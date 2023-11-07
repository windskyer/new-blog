---
title: docker看veth对
categories: Kubernetes
sage: false
date: 2020-04-01 17:40:13
tags:
---

## 找到网卡对应的方式，在主机上执行如下命令

```bash
docker exec -it <container-name> bash -c 'cat /sys/class/net/eth0/iflink'

# 假设返回 12
grep -l 12 /sys/class/net/veth*/ifindex
# 此时会有如下类似返回
/sys/class/net/veth11d4238/ifindex
# veth11d4238 即主机上的另一半
```
<!-- more -->

做成一个脚本 vethfinder

```bash
#!/bin/bash
for container in $(docker ps -q); do
    iflink=`docker exec -it $container bash -c 'cat /sys/class/net/eth0/iflink'`
    iflink=`echo $iflink|tr -d '\r'`
    veth=`grep -l $iflink /sys/class/net/veth*/ifindex`
    veth=`echo $veth|sed -e 's;^.*net/\(.*\)/ifindex$;\1;'`
    echo $container:$veth
done
```

执行

```bash
$ docker ps -q
c4d8096eff43
34ac6e9f1e6e
d5a2aa5f3de3
 
$ sudo ./vethfinder
c4d8096eff43:veth11d4238
34ac6e9f1e6e:veth7d52cd1
d5a2aa5f3de3:vethe46073d
```