---
title: pod访问svc不通问题
categories: Kubernetes
sage: false
date: 2019-12-16 15:43:26
tags: flannel
---

## 集群环境

k8s: v.1.15.3
cni: flannel
proxy: ipvs

<!-- more -->

## 网络访问情景

### pod 访问自身svc

pod A -> svc -> pod A

原因: 开启 网桥"hairpinMode" 默认

解决：cni config文件中配置

```yaml
 cni-conf.json: |
    {
      "name":"cni0",
      "cniVersion":"0.3.1",
      "plugins":[
        {
          "type":"flannel",
          "delegate":{
            "forceAddress":true,
            "isDefaultGateway":true,
            "hairpinMode":true
          }
        },
        {
```

### pod 访问通node节点上pod的svc

pod A -> svc -> pod B (pod A 和 pod B 在同一node节点上同一cni0 网桥内)

原因： net.bridge.bridge-nf-call-arptables net.bridge.bridge-nf-call-ip6tables 没开启

解决：sysctl -w net.bridge.bridge-nf-call-arptables = 1
      sysctl -w net.bridge.bridge-nf-call-ip6tables = 1

### pod 访问https的svc(LoadBalancer 类型)

pod A(https) -> lb(公网ip) -> pod B

原因： prxoy 给iptables 添加转发规则，导致集群内部访问svc的公网ip会直接转给相应的pod ip

解决：把tls证书放在ingress 上。
