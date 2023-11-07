---
title: etcd重置
categories: Kubernetes
sage: false
date: 2020-05-18 10:38:58
tags: etcd
---

## 获取现有etcd集群状态

```bash
ETCDCTL_API=3 etcdctl --endpoints=33.41.0.187:3379 -w table endpoint status --cluster #查看集群leader 节点
```

<!-- more -->

## 重新选择etcd集群leader

```bash
ETCDCTL_API=3 etcdctl --endpoints=33.20.11.65:3379 -w table endpoint status --cluster #查看etcd 集群状态
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://33.20.21.192:3379 | 4829fde917c40977 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
| http://33.20.21.128:3379 | 4b1f65443139e885 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
| http://33.20.11.34:3379  | afa7e820d9efe9e3 |  3.2.24 |  9.2 MB |      true |      false |         2 |     696565 |                  0 |        |
| http://33.20.11.65:3379  | bd245184166d442c |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
| http://33.20.11.85:3379  | d41c98da852f899e |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

ETCDCTL_API=3 etcdctl --endpoints=33.20.11.34:3379  move-leader  d41c98da852f899e # 转移leader权限给d41c98da852f899e  节点
# 注意： --endpoints=33.20.11.34:3379 必须指定leader节点地址

```

## etcd集群移除member节点

```bash
ETCDCTL_API=3 etcdctl --endpoints=33.20.11.65:3379 -w table endpoint status --cluster #查看etcd 集群状态
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://33.20.21.192:3379 | 4829fde917c40977 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
| http://33.20.21.128:3379 | 4b1f65443139e885 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.34:3379 | afa7e820d9efe9e3 |  3.2.24 |  9.2 MB |      true |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.65:3379 | bd245184166d442c |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.85:3379 | d41c98da852f899e |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

ETCDCTL_API=3 etcdctl --endpoints=33.20.11.34:3379 member remove d41c98da852f899e # 移除 d41c98da852f899e  节点
```

## etcd集群添加member节点

```bash
ETCDCTL_API=3 etcdctl --endpoints=33.20.11.65:3379 -w table endpoint status --cluster #查看etcd 集群状态
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://33.20.21.192:3379 | 4829fde917c40977 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
| http://33.20.21.128:3379 | 4b1f65443139e885 |  3.2.24 |  9.2 MB |     false |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.34:3379 | afa7e820d9efe9e3 |  3.2.24 |  9.2 MB |      true |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.65:3379 | bd245184166d442c |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
|  http://33.20.11.85:3379 | d41c98da852f899e |  3.2.24 |  9.3 MB |     false |      false |         2 |     696565 |                  0 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
 
# 删除etcd 脏数据  - --initial-cluster-state=existing  - --initial-cluster=eventetcd-33.20.11.34=http://33.20.11.34:5001,eventetcd-33.20.11.85=http://33.20.11.85:5001,eventetcd-33.20.11.65=http://33.20.11.65:5001
rm -fr /data/etcd/eventetcd-33.20.11.65
```
### 修改etcd 配置项
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: eventetcd
    tier: control-plane
  name: eventetcd
  namespace: kube-system
spec:
  containers:
  - command:
    - /usr/local/bin/etcd
    - --name=eventetcd-33.20.11.65
    - --data-dir=/data/etcd/eventetcd-33.20.11.65
    - --snapshot-count=10000
    - --heartbeat-interval=10000
    - --election-timeout=50000
    - --max-snapshots=5
    - --listen-peer-urls=http://33.20.11.65:5001
    - --listen-client-urls=http://127.0.0.1:3379,http://33.20.11.65:3379
    - --initial-advertise-peer-urls=http://33.20.11.65:5001
    - --initial-cluster=eventetcd-33.20.11.34=http://33.20.11.34:5001,eventetcd-33.20.11.85=http://33.20.11.85:5001,eventetcd-33.20.11.65=http://33.20.11.65:5001
    - --initial-cluster-state=existing
    - --initial-cluster-token=eventetcd-cluster
    - --advertise-client-urls=http://33.20.11.65:3379
    - --auto-compaction-retention=1
    - --max-request-bytes=10485760
    - --quota-backend-bytes=8589934592
    image: etcd:3.4.14
```

### 加入现有etcd集群

```bash
ETCDCTL_API=3 etcdctl --endpoints=33.20.11.34:3379 member add eventetcd-33.20.11.65 --peer-url http://33.20.11.65:5001

# 启动event etcd服务或者pod
```

## 现有event etcd集群的节点加入到新event etcd集群

>1. 在老集群中移除event etcd节点
>2. 停止现有event etcd 的服务或者pod
>3. 在新集群中添加event etcd节点
