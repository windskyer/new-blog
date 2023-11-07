---
title: pod eviction
categories: Kubernetes
sage: false
date: 2019-04-10 20:13:28
tags: pod
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# K8S 的 pod eviction

K8S 有个特色功能叫 pod eviction，它在某些场景下如节点 NotReady，资源不足时，把 pod 驱逐至其它节点。本文首先介绍该功能，最后谈谈落地经验。

<!-- more -->

## 介绍

从发起模块的角度，pod eviction 可以分为两类：

- Kube-controller-manager: 周期性检查所有节点状态，当节点处于 NotReady 状态超过一段时间后，驱逐该节点上所有 pod。
- Kubelet: 周期性检查本节点资源，当资源不足时，按照优先级驱逐部分 pod。

## Kube-controller-manger 发起的驱逐

Kube-controller-manager 周期性检查节点状态，每当节点状态为 NotReady，并且超出 podEvictionTimeout 时间后，就把该节点上的 pod 全部驱逐到其它节点，其中具体驱逐速度还受驱逐速度参数，集群大小等的影响。最常用的 2 个参数如下：

- –pod-eviction-timeout：NotReady 状态节点超过该时间后，执行驱逐，默认 5 min。
- –node-eviction-rate：驱逐速度，默认为 0.1 pod/秒

当某个 zone 故障节点的数目超过一定阈值时，采用二级驱逐速度进行驱逐。

- –large-cluster-size-threshold：判断集群是否为大集群，默认为 50，即 50 个节点以上的集群为大集群。
- –unhealthy-zone-threshold：故障节点数比例，默认为 55%
- –secondary-node-eviction-rate：当大集群的故障节点超过 55% 时，采用二级驱逐速率，默认为 0.01 pod／秒。当小集群故障节点超过 55% 时，驱逐速率为 0 pod／秒。

判断是否驱逐的代码如下：

```go
func (nc *NodeController) monitorNodeStatus() error {
        ......
        if currentReadyCondition != nil {
            // Check eviction timeout against decisionTimestamp
            if observedReadyCondition.Status == api.ConditionFalse &&
                decisionTimestamp.After(nc.nodeStatusMap[node.Name].readyTransitionTimestamp.Add(nc.podEvictionTimeout)) {
                if nc.evictPods(node) {
                    glog.V(2).Infof("Evicting pods on node %s: %v is later than %v + %v", node.Name, decisionTimestamp, nc.nodeStatusMap[node.Name].readyTransitionTimestamp, nc.podEvictionTimeout)
                }
            }
            if observedReadyCondition.Status == api.ConditionUnknown &&
                decisionTimestamp.After(nc.nodeStatusMap[node.Name].probeTimestamp.Add(nc.podEvictionTimeout)) {
                if nc.evictPods(node) {
                    glog.V(2).Infof("Evicting pods on node %s: %v is later than %v + %v", node.Name, decisionTimestamp, nc.nodeStatusMap[node.Name].readyTransitionTimestamp, nc.podEvictionTimeout-gracePeriod)
                }
            }
}
```

## Kubelet 发起的驱逐

Kubelet 周期性检查本节点的内存和磁盘资源，当可用资源低于阈值时，则按照优先级驱逐 pod，具体检查的资源如下：

- memory.available
- nodefs.available
- nodefs.inodesFree
- imagefs.available
- imagefs.inodesFree

以内存资源为例，当内存资源低于阈值时，驱逐的优先级大体为 BestEffort > Burstable > Guaranteed，具体的顺序可能因实际使用量有所调整。当发生驱逐时，kubelet 支持 soft 和 hard 两种模式，soft 模式表示缓期一段时间后驱逐，hard 模式表示立刻驱逐。

## 落地经验

对于 kubelet 发起的驱逐，往往是资源不足导致，它优先驱逐 BestEffort 类型的容器，这些容器多为离线批处理类业务，对可靠性要求低。驱逐后释放资源，减缓节点压力，弃卒保帅，保护了该节点的其它容器。无论是从其设计出发，还是实际使用情况，该特性非常 nice。

对于由 kube-controller-manager 发起的驱逐，效果需要商榷。正常情况下，计算节点周期上报心跳给 master，如果心跳超时，则认为计算节点 NotReady，当 NotReady 状态达到一定时间后，kube-controller-manager 发起驱逐。然而造成心跳超时的场景非常多，例如：

- 原生 bug：kubelet 进程彻底阻塞
- 误操作：误把 kubelet 停止
- 基础设施异常：如交换机故障演练，NTP 异常，DNS 异常
- 节点故障：硬件损坏，掉电等

从实际情况看，真正因计算节点故障造成心跳超时概率很低，反而由原生 bug，基础设施异常造成心跳超时的概率更大，造成不必要的驱逐。

理想的情况下，驱逐对无状态且设计良好的业务方影响很小。但是并非所有的业务方都是无状态的，也并非所有的业务方都针对 Kubernetes 优化其业务逻辑。例如，对于有状态的业务，如果没有共享存储，异地重建后的 pod 完全丢失原有数据；即使数据不丢失，对于 Mysql 类的应用，如果出现双写，重则破坏数据。对于关心 IP 层的业务，异地重建后的 pod IP 往往会变化，虽然部分业务方可以利用 service 和 dns 来解决问题，但是引入了额外的模块和复杂性。

除非满足如下需求，不然请尽量关闭 kube-controller-manager 的驱逐功能，即把驱逐的超时时间设置非常长，同时把一级／二级驱逐速度设置为 0。否则，非常容易就搞出大大小小的故障，血泪的教训。

- 业务方要用正确的姿势使用容器，如数据与逻辑分离，无状态化，增强对异常处理等
- 分布式存储
- 可靠的 Service／DNS 服务或者保持异地重建后的 IP 不变

通常情况下，从架构设计的角度来说，管理模块的不可用并不会影响已有的实例。例如：OpenStack 计算节点的服务异常，管理节点不会去变动该节点上已经运行虚拟机。但是 kubernetes 的 pod eviction 则打破了这种设计，出发点很新颖，但是落地时，需要充分的考虑业务方的特点，以及业务方的设计是否充分考虑了 kubernetes 的特色。

## 参考

[Configure Out Of Resource Handling](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)
[记一次 k8s 集群单点故障引发的血案](https://blog.csdn.net/YueDaoShengSi/article/details/54883039)
[容器化RDS——计算存储分离架构下的“Split-Brain”](https://mp.weixin.qq.com/s?__biz=MzA5OTAyNzQ2OA==&mid=2649696704&idx=1&sn=5714e324b212b0eb3b862f8600b345fa&chksm=889316a3bfe49fb5e8db58aa595e0e4a74556e050be1b8894c60ee7e00fe91abfdd28d97412d&scene=21#wechat_redirect)