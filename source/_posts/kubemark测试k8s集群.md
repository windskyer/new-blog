---
title: kubemark测试k8s集群
categories: Kubernetes
sage: false
date: 2020-01-07 15:38:42
tags:
---

## 搭建 kubemark 流程

由于官方搭建的 kubemark 集群依赖比较多且不是特别符合我们的场景，所以以下针对金上云的集群环境搭建测试集群。

先使用金山云平台搭建两个真实的集群。集群规格是 3 master(2c4g), 2 node(32c64g)，一个作为 kubemark 集群， 另一个用来部署 hollow-node pod，称为 support 集群。

配置本地 kubectl 连接到support集群。

在support集群创建如下资源

## 集群配置

测试集群：
    master个数： 3
    master规格： 1c2g ~ 2c4g
    node： 使用kubemark模拟

<!-- more -->

一共测试了以下10个case：

>1. 50 node
>2. 100 node
>3. 150 node
>4. 200 node
>5. 250 node
>6. 300 node
>7. 350 node
>8. 400 node
>9. 450 node
>10. 500 node
