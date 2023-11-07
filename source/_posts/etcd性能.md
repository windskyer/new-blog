---
title: etcd性能测试
categories: Kubernetes
sage: false
date: 2019-12-27 14:13:34
tags: etcd
---


## 理解性能

决定 etcd 性能的关键因素，包括：

>1. 延迟(latency)：延迟是完成操作的时间。
>2. 吞吐量(throughput)：吞吐量是在某个时间期间之内完成操作的总数量。 当 etcd 接收并发客户端请求时，通常平均延迟随着总体吞吐量增加而增加。

<!-- more -->

etcd 使用 Raft 一致性算法来在成员之间复制请求并达成一致。

一致性性能，特别是提交延迟，受限于两个物理约束：

>1. 网络IO延迟
>2. 磁盘IO延迟

完成一个 etcd 请求的最小时间是成员之间的网络往返时延(Round Trip Time / RTT)，加需要提交数据到持久化存储的 fdatasync 时间。

>1. 在一个数据中心内的 RTT 可能有数百毫秒。

机械硬盘的典型 fdatasync 延迟是大概 10ms。对于 SSD 硬盘, 延迟通常低于 1ms。

为了提高吞吐量, etcd 将多个请求打包在一起并提交给 Raft。

>1. 这个批量策略让 etcd 在重负载试获得高吞吐量。

有其他子系统影响到 etcd 的整体性能。

>1. 每个序列化的 etcd 请求必须通过 etcd 的 boltdb支持的(boltdb-backed) MVCC 存储引擎,它通常需要10微秒来完成。

etcd 定期递增快照它最近实施的请求，将他们和之前在磁盘上的快照合并。这个过程可能导致延迟尖峰(latency spike)。

>1. 虽然在SSD上这通常不是问题，在HDD上它可能加倍可观察到的延迟。

进行中的压缩可以影响 etcd 的性能。

>1. 幸运的是，压缩通常无足轻重，因为压缩是错开的，因此它不和常规请求竞争资源。

RPC 系统，gRPC，为 etcd 提供定义良好，可扩展的 API。

>1. 但是它也引入了额外的延迟，尤其是本地读取。

## 评测性能

可以通过 etcd 自带的 [benchmark](https://github.com/coreos/etcd/tree/master/tools/benchmark) CLI 工具来评测 etcd 的性能。

对于某些基线性能数字，考虑搭建3节点 etcd 集群，带有下列硬件配置：

>1. 搭建 etcd 集群：3 副本etcd pod， 1 vCPUs + 2GB Memory + ssd磁盘（每个机器都有不少进程在上面）
>2. 评测 etcd 节点：1 台机器(客户端)，16 vCPUs + 32GB Memory + 普通磁盘，用于做benchmark
>3. 操作系统：Centos 7.5
>4. 容器镜像：quay.io/coreos/etcd:3.3.10

对应的pod ip列表如下：

| 机器名 | IP |
|-------|--------------|
| pod 1 | 10.245.29.16 |
| pod 2 | 10.245.21.35 |
| pod 3 | 10.245.17.55 |
| CLIENT | 10.246.12.0/24 |

使用这些配置，采样命令如下:

```sh
HOST_1=http://10.245.29.16:2379  
HOST_2=http://10.245.21.35:2379  
HOST_3=http://10.245.17.55:2379

# 假定 HOST_1 是 leader, 写入请求发到 leader，连接数1，客户端数1
benchmark --endpoints=${HOST_1} --conns=1 --clients=1 put --key-size=8 --sequential-keys --total=10000 --val-size=256

# 假定 HOST_1 是 leader, 写入请求发到 leader，连接数100，客户端数1000
benchmark --endpoints=${HOST_1} --conns=100 --clients=1000 put --key-size=8 --sequential-keys --total=100000 --val-size=256

# 写入发到所有成员，连接数100，客户端数1000
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 put --key-size=8 --sequential-keys --total=100000 --val-size=256
```

etcd 近似的写入指标如下：

|key的数量 | Key的大小 | Value的大小| 连接数量 | 客户端数量 | 目标 etcd 服务器 | 平均写入 QPS | 每请求平均延迟 | 内存 |
|-------- |-----------|-----------|---------|------------|-----------------|-------------|---------------|-----|
| 10,000  | 8         | 256       | 1       | 1          | 只有主           | 126         | 7.7ms         |     |
| 100,000 | 8         | 256       | 100     | 1000       | 只有主           | 16,033      | 60ms          |     |
| 100,000 | 8         | 256       | 100     | 1000       | 所有成员         | 15,102      | 63ms          |     |

**注：key和value的大小单位是 字节 / bytes**

为了一致性，线性化(Linearizable)读取请求要通过集群成员的法定人数来获取最新的数据。串行化(Serializable)读取请求比线性化读取要廉价一些，因为他们是通过任意单台 etcd 服务器来提供服务，而不是成员的法定人数，代价是可能提供过期数据。

采样命令如下:

```sh
HOST_1=http://10.245.29.16:2379  
HOST_2=http://10.245.21.35:2379  
HOST_3=http://10.245.17.55:2379

# Linearizable 读取请求
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=1 --clients=1 range foo --consistency=l --total=10000
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 range foo --consistency=l --total=100000

# Serializable 读取请求，使用每个成员然后将数字加起来
for endpoint in ${HOST_1} ${HOST_2} ${HOST_3}; do  
    benchmark --endpoints=$endpoint --conns=1 --clients=1 range foo --consistency=s --total=10000
done  
for endpoint in ${HOST_1} ${HOST_2} ${HOST_3}; do  
    benchmark --endpoints=$endpoint --conns=100 --clients=1000 range foo --consistency=s --total=100000
done  
```

etcd 近似读取指标如下：

| 请求数量 | Key 大小 | Value 大小 | 连接数量 | 客户端数量 | 一致性 | 每请求平均延迟 | 平均读取 QPS |
|---------|----------|-----------|---------|------------|--------|--------------|--------------|
| 10,000  | 8        | 256       | 1        | 1         | 线性化 | 1.2ms         | 824         |
| 10,000  | 8        | 256       | 1        | 1         | 串行化 | 0.6ms         | 1,628       |
| 100,000 | 8        | 256       | 100      | 1000      | 线性化 | 23.8ms        | 37,679      |
| 100,000 | 8        | 256       | 100      | 1000      | 串行化 | 43.5ms        | 22,458      |

## 分析

读取指标的时候，按理说串行化要比线性化要好

影响性能的因素：磁盘（是否SSD），CPU（很多其他应用），内存（这个相对比较充足）
