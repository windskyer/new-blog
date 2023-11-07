---
title: k8s集群支持ipv6
categories: Kubernetes
sage: false
date: 2020-08-19 17:46:28
tags: k8s
---

## 预备环境

kubernetes 从1.16版本开始支持 ipv4/ipv6 双栈，开启 ipv4/ipv6 后，集群pod都会分配ipv4和ipv6两个地址，service 会根据配置的 ipFamily 类型来确定分配的集群ip类型。而且从pod出集群的网络会同时走ipv4和ipv6接口

要支持ipv4/ipv6，需要满足一些必备的要求:
>- kubernetes 版本 1.16 以上
>- 部署集群的云主机支持ipv4/ipv6
>- 网络插件需要支持 ipv4/ipv6
>- kube-proxy 需要运行在 ipvs 模式
<!-- more -->

## 开启ipv4/ipv6

***注意: 创建集群时，如果要开启ipv6，则集群所在vpc必须开启ipv6，云主机也必须支持开启 ipv6网络，集群版本需要大于等于 v1.16 ***

### 集群服务配置

kube-controller-manager
```bash
--feature-gates="IPv6DualStack=true"
--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR> eg. --cluster-cidr=10.244.0.0/16
--service-cluster-ip-range=<IPv4 CIDR>, <IPv6 CIDR>
--node-cidr-mask-size-ipv4 |  --node-cidr-mask-size-ipv6  defaults to /24 for IPv4 and /64 for IPv6
```

kubelet
```bash
--feature-gates="IPv6DualStack=true"
```

kube-proxy
```bash

--proxy-mode=ipvs
--cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>
--feature-gates="IPv6DualStack=true"
```

### 网络插件

Kubenet or Calico 已经支持 ipv6，flannel还没有支持[issue](https://github.com/coreos/flannel/issues/248)
calico 开启ipv6：[Enabling IPv6 Support](https://docs.projectcalico.org/networking/ipv6)

## Services

一个 service 只会分配一个ip，所在在开启ipv4/ipv6双网络栈的集群中，创建service时需要指定分配的ip类型, 配置  .spec.ipFamily 字段
>- ipv4: 将会给service分配ipv4的cluster ip
>- ipv6: 将会给service分配ipv6的cluster ip

## 负载均衡

目前默认只会创建ipv4的负载均衡，集群开启ipv6后，需要可以根据服务的类型创建ipv6的负载均衡
