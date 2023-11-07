---
title: coredns
categories: Kubernetes
sage: false
date: 2019-10-10 14:40:50
tags: dns
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

## coredns 启动失败问题

coredns 在ubuntu 下启动报错

```sh
plugin/loop: Loop (127.0.0.1:55953 -> :1053) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 4547991504243258144.3688648895315093531."
```

<!-- more -->

原因： ubuntu系统下的/etc/resolv.conf 中的地址是回环地址127.0.0.53 而在coredns pod的网络和宿主机的网络是不同的。 在ubuntu 宿主机上通过回环地址127.0.0.53  可以指向真实的dns地址，
而在coredns pod中在指向的pod 本地地址，无法获取真实的dns地址

解决方法：

Add the following to your kubelet config yaml: resolvConf: <path-to-your-real-resolv-conf-file> (or via command line flag --resolv-conf deprecated in 1.10). Your “real” resolv.conf is the one that contains the actual IPs of your upstream servers, and no local/loopback address. This flag tells kubelet to pass an alternate resolv.conf to Pods. For systems using systemd-resolved, /run/systemd/resolve/resolv.conf is typically the location of the “real” resolv.conf, although this can be different depending on your distribution.

启动kubelet 时候加入 --resolv-conf 参数 根据不同的操作系统指定真实的dns 值

ubuntu： --resolv-conf=/run/systemd/resolve/resolv.conf
centos：  --resolv-conf=/etc/resolv.conf