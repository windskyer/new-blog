---
title: k8s面试
categories: Kubernetes
sage: true
hidden: true
date: 2021-12-30 00:46:33
tags: 面试
---

## cgroup 原理简介
cgroups 是Linux内核提供的一种可以限制单个进程或者多个进程所使用资源的机制，可以对 cpu，内存等资源实现精细化的控制，目前越来越火的轻量级容器 Docker 就使用了 cgroups 提供的资源限制能力来完成cpu，内存等部分的资源控制。

cpu 子系统，主要限制进程的 cpu 使用率。
cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
memory 子系统，可以限制进程的 memory 使用量。
blkio 子系统，可以限制进程的块设备 io。
devices 子系统，可以控制进程能够访问某些设备。
net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。
freezer 子系统，可以挂起或者恢复 cgroups 中的进程。
ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace。

## grpc原理
RPC 的核心功能主要由 5 个模块组成，如果想要自己实现一个 RPC，最简单的方式要实现三个技术点，分别是：
>1. 服务寻址
>2. 数据流的序列化和反序列化
>3. 网络传输

## RPC需要解决的三个问题
>1. Call ID映射。我们怎么告诉远程机器我们要调用哪个函数呢？在本地调用中，函数体是直接通过函数指针来指定的，我们调用具体函数，编译器就自动帮我们调用它相应的函数指针。但是在远程调用中，是无法调用函数指针的，因为两个进程的地址空间是完全不一样。所以，在RPC中，所有的函数都必须有自己的一个ID。这个ID在所有进程中都是唯一确定的。客户端在做远程过程调用时，必须附上这个ID。然后我们还需要在客户端和服务端分别维护一个 {函数 <--> Call ID} 的对应表。两者的表不一定需要完全相同，但相同的函数对应的Call ID必须相同。当客户端需要进行远程调用时，它就查一下这个表，找出相应的Call ID，然后把它传给服务端，服务端也通过查表，来确定客户端需要调用的函数，然后执行相应函数的代码。
>2. 序列化和反序列化。客户端怎么把参数值传给远程的函数呢？在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。但是在远程过程调用时，客户端跟服务端是不同的进程，不能通过内存来传递参数。甚至有时候客户端和服务端使用的都不是同一种语言（比如服务端用C++，客户端用Java或者Python）。这时候就需要客户端把参数先转成一个字节流，传给服务端后，再把字节流转成自己能读取的格式。这个过程叫序列化和反序列化。同理，从服务端返回的值也需要序列化反序列化的过程。
>3. 网络传输。远程调用往往是基于网络的，客户端和服务端是通过网络连接的。所有的数据都需要通过网络传输，因此就需要有一个网络传输层。网络传输层需要把Call ID和序列化后的参数字节流传给服务端，然后再把序列化后的调用结果传回客户端。只要能完成这两者的，都可以作为传输层使用。因此，它所使用的协议其实是不限的，能完成传输就行。尽管大部分RPC框架都使用TCP协议，但其实UDP也可以，而gRPC干脆就用了HTTP2。Java的Netty也属于这层的东西。

### 内核态和用户态通行方式
一般是在用户空间，通过系统调用函数来访问内核空间
除此之外，还有以下四种方式：

procfs(/proc) 是 进程文件系统 的缩写，它本质上是一个伪文件系统，为什么说是 伪 文件系统呢？因为它不占用外部存储空间，只是占用少量的内存，通常是挂载在 /proc 目录下。
sysctl(/proc/sys) 它使用的是 /proc 的一个子目录 /proc/sys。和 procfs 的区别在于：procfs 主要是输出只读数据，而 sysctl 输出的大部分信息是可写的。
sysfs(/sys)是 Linux 2.6 才引入的一种虚拟文件系统，它的做法也是通过文件 /sys 来完成用户态和内核的通信。和 procfs 不同的是，sysfs 是将一些原本在 procfs 中的，关于设备和驱动的部分，独立出来，以 “设备树” 的形式呈现给用户。
netlink 套接口 netlink 是 Linux 用户态与内核态通信最常用的一种方式。Linux kernel 2.6.14 版本才开始支持。它本质上是一种 socket，常规 socket 使用的标准 API，在它身上同样适用。比如创建一个 netlink socket，可以调用如下的 socket 函数：

### 简介
gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计。目前提供 C、Java 和 Go 语言版本，分别是：grpc, grpc-java, grpc-go. 其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP 和 C# 支持。

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特性。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。

### 调用模型
>1. 客户端（gRPC Stub）调用 A 方法，发起 RPC 调用。
>2. 对请求信息使用 Protobuf 进行对象序列化压缩（IDL）。
>3. 服务端（gRPC Server）接收到请求后，解码请求体，进行业务逻辑处理并返回。
>4. 对响应结果使用 Protobuf 进行对象序列化压缩（IDL）。
>5. 客户端接受到服务端响应，解码请求体。回调被调用的 A 方法，唤醒正在等待响应（阻塞）的客户端调用并返回响应结果。

<!-- more -->
## etcd 的一致性协议raft

核心使用了RAFT分布式一致性协议。一致性这个概念，它是指多个服务器在状态达成一致。

>1. leader 处理所有客户端交互，日志复制等，一个任期只有一个。
>2. follower 完全被动的选民，是只读的。
>3. candidate 候选人，可以被选举为新领导。

选举流程：

1. 首先是当3台节点启动后都是follower
2. 当这些follower 没有收到leader 心跳，超时后会开始选举投票
3. 收到其余大多数follower 的选票后成为leader

选举正确性：
>1. 在每一任期内，最多允许一个服务被选举为leader
>2. 在一个任期内，一个服务只能投一票
>3. 只有获得大多数投票才能作为leader
>4. 如果有多个candidate，最终一定会有一个被选举为leader
>5. 如果多个candidate同时发起了选举，导致都没有获得大多数选票时，每一个candidate会随机等待一段时间后重新发起新一轮投票（一般是随机等待150-300ms）

接受请求：
>1. Leader接受到客户端写请求后，会将数据更新写入到log中
>2. 如果S2和S3收到客户端写请求，会将请求转发到Leader S1
>3. Leader会异步的将更新的log同步到Follower S2和S3
>4. 超过多数的Follower将数据成功同步到log后，Leader会将该条数据更新为Committed状态，Committed index会随着增长。


优化方案：
boltdb 使用新的 hashmap 的方式进行数据查找 旧的算法查找arraymap
etcd 的磁盘优先级 ionice
tc 命令控制对应端口2380 的流量


## Controller Manager作为集群内部的管理控制中心 介绍

Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

>1. Replication Controller
>2. Node Controller
>3. CronJob Controller
>4. Daemon Controller
>5. Deployment Controller
>6. Endpoint Controller
>7. Garbage Collector
>8. Namespace Controller


## k8s 中 informer 简介

Informer依赖Kubernetes的List/Watch API。 通过Lister()对象来List/Get对象时，Informer不会去请求Kubernetes API，而是直接查询本地缓存，减少对Kubernetes API的直接调用。
Informer 只会调用 Kubernetes List 和 Watch 两种类型的 API。Informer 在初始化的时，先调用 Kubernetes List API 获得某种 resource 的全部 Object，缓存在内存中; 然后，调用 Watch API 去 watch 这种 resource，去维护这份缓存; 最后，Informer 就不再调用 Kubernetes 的任何 API。

Reflector：通过Kubernetes Watch API监听resource下的所有事件，list 获取所有资源，watch获取指定资源的变更事件
Lister：用来被调用List/Get方法
Controller: 负责管理事件处理函数
Processor：记录并触发回调函数
DeltaFIFO： 先进先出队列
LocalStore：内存缓存

>1. Informer 在初始化时，Reflector 会先 List API 获得所有的 Pod
>2. Reflect 拿到全部 Pod 后，会将全部 Pod 放到 Store 中
>3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据
>4. Informer 初始化完成之后，Reflector 开始 Watch Pod，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件
>5. Reflector 将 pod_1 被删除 的这个事件发送到 DeltaFIFO
>6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1
>7. DeltaFIFO 再 Pop 这个事件到 Controller 中
>8. Controller 收到这个事件，会触发 Processor 的回调函数
>9. LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中

## k8s 中 list&watch 简介

它采用统一的异步消息处理机制，保证了消息的实时性、可靠性、顺序性和性能等。
list-watch本质上还是client端监听k8s资源变化并作出相应处理的生产者消费者框架。
list就是获取静态的所有数据，而watch则是只关心发生了变化的那部分。
list非常好理解，就是调用资源的list API罗列资源，基于HTTP短链接实现；watch则是调用资源的watch API监听资源变更事件，基于HTTP 长链接实现，也是本文重点分析的对象。
http 长连接实现基于 HTTP 分块传输编码机制实现的

## helm 搭建

Chart: 就是 Helm 的一个包（package），包含一个应用所有的 Kubernetes manifest 模版，类似于 YUM 的 RPM 或者 APT 的 dpkg 文件。
Helm CLI: Helm 的客户端组件，它通过 gRPC aAPI 向 tiller 发送请求。
Tiller: Helm 的服务器端组件，在 Kubernetes 群集上运行，负载解析客户端端发送过来的 Chart，并根据 Chart 中的定义在 Kubernetes 中创建出相应的资源，tiller 把 release 相关的信息存入 Kubernetes 的 ConfigMap 中。
Repository: 用于发布和存储 Chart 的仓库。
Release: 可以理解成 Chart 部署的一个实例。通过 Chart 在 Kubernetes 中部署的应用都会产生一个唯一的 Release，即使是同一个 Chart，部署多次就会产生多个 Release。

## kube-schedulable 调度简介

算法需要经过两个阶段，分别是过滤和打分，首先过滤掉一部分，保证剩余的节点都是可调度的，接着在打分阶段选出最高分节点，该节点就是scheduler的输出节点。

过滤节点
// 默认过滤规则14个
NoVolumeZoneConflictPred
MaxEBSVolumeCountPred
MaxGCEPDVolumeCountPred
MaxAzureDiskVolumeCountPred
MaxCSIVolumeCountPred
MatchInterPodAffinityPred
NoDiskConflictPred
GeneralPred
CheckNodeMemoryPressurePred： 检查mem
CheckNodeDiskPressurePred： 检查磁盘
CheckNodePIDPressurePred： 检查pid
CheckNodeConditionPred： 节点状态ready，磁盘足够，网络是否正确配置，是否可以调度unschedulable: true( kubectl cordon node 设置不允许调度/uncordon允许调度)
PodToleratesNodeTaintsPred
CheckVolumeBindingPred：检查对应pvc 是否绑定，pv是否挂载到对应node上

计算权重
SelectorSpreadPriority： 相同service／rc的pods越分散，得分越高， 不通zone的node 分数越高
InterPodAffinityPriority：
NodeAffinityPriority：
TaintTolerationPriority：


## Istio 服务网格

Istio 服务网格从逻辑上分为数据平面和控制平面：

>1. 数据平面：有代理（Envoy）组成，以sidecar方式部署在用户业务pod中，共享pod的网络命名空间，通过iptables redirect的方式将原本进入用户容器的流量转发Envoy 代理；Envoy代理负责控制服务间网络通信
>2. 控制平面：按需制定流量规则，并下发给代理来实现对流量的控制

https://www.flftuu.com/2020/09/13/Istio-%E6%9E%B6%E6%9E%84%E4%B8%8E%E6%A6%82%E5%BF%B5/