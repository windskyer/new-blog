---
title: k8s容器固定ip
categories: Kubernetes
sage: false
date: 2019-11-06 11:03:48
tags: docker
---

## 需求

kubernetes集群支持容器IP不变（具体来讲，是通过Statefulset创建出来的pod，保持IP不变），
固定IP的使用场景是这样的：在开发测试环境中，给开发人员分配一个容器，开发人员通过容器的IP ssh登录进去，进行开发调试。
由于这个环境开发人员会长期使用，因此希望容器在发生漂移、重启等变动时，仍然能通过原来的IP登录上去。

<!-- more -->

## 方案

IP分配的功能是CNI组件实现的，如flannel作为网络模型，其容器IP分配是和node绑定的（每个node分配的IP都是一个固定网段的）。当容器从一个节点漂移另一个节点后，无法保持原来的IP。

考虑基于calico网络模型来实现。关于calico的介绍及实现
参考：
    [容器网络-从CNI到Calico](http://dockone.io/article/2578)
    [calico代码解读](https://chenxy.blog.csdn.net/article/details/75270578)

calico支持两种类型的ipam，host-local和calico-ipam。host-local的ip分配方式，也是和node节点绑定的。calico-ipam，可以在用户指定的整个容器地址网段内分配IP地址。

分析calico-ipam的代码，发现其结构清晰，易于改造。
实现固定IP的思路如下：
    分配IP时，检查当前pod是不是statefulset创建出来的。如果是，检查configmap中有没有保存过pod名字和ip的对应关系，如果有，跳转到指定IP分配逻辑。如果没有，转入calico-ipam的自动IP分配逻辑，并在结束时将pod名字和ip的对应关系，记录到configmap中。
