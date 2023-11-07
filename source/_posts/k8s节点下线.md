---
title: k8s节点下线
categories: Kubernetes
sage: false
date: 2020-04-20 16:43:04
tags: k8s
---

下线一般有两种情况,一般是故障或者是迁移。故障节点下线只需要直接摘除下来就可以，因为会从新调度到新的节点。而正常节点迁移则需要先排干节点，即将所有pod在此节点上迁移出去其他节点。
<!-- more -->
# 正常节点下线

## 查看节点
```bash
kubectl get node xxx
```
## 排干,排干时他们提示忽略了ds的pod

```bash
kubectl drain hdss7-21.host.com --delete-local-data --force --ignore-daemonsets
```

**注意: 在排干的过程中,此节点标记为不可以调度的状态**

## 删除节点
```bash
kubectl delete node hdss7-21.host.com
```

异常节点直接删除即可，修复后启动服务，节点将自动加入集群
# 异常节点下线
## 删除节点
```bash
kubectl delete node hdss7-21.host.com
```