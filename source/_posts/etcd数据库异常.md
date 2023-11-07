---
title: etcd数据库异常
categories: Kubernetes
sage: false
date: 2020-05-03 20:10:46
tags: etcd
---


# Etcd 维护

Etcd 集群少不了日常维护来保持其可用性。这些运维操作一般都是自动化且期间 Etcd 不会停止对外服务，或者严重影响 Etcd 集群的性能。

所有的运维管理都在操作 Etcd 的存储空间。存储空间的配额用于控制 Etcd 数据空间的大小，如果 Etcd 节点磁盘空间不足了，配额会触发告警，然后 Etcd 系统将进入操作受限的维护模式。为了避免存储空间消耗完导致写不进去，应该定期清理 key 的历史版本。在清理 Etcd 节点存储碎片之后，存储空间会重新进行调整。最后，定期对 Etcd 节点状态做快照备份，以便在错误的运维操作引起数据丢失或数据不一致时进行数据恢复。
<!-- more -->

## 压缩历史版本

对于 Etcd 为每个 key 都保存了历史版本，因此这些历史版本需要进行周期性地压缩，以避免出现性能问题或存储空间耗尽的问题。压缩历史版本会丢弃该 key 给定版本之前的所有信息，节省出来的空间可以用于后续的写操作。

key 的历史版本可以通过 Etcd 带时间窗口的历史版本来保留策略自动压缩，或通过 etcdctl 命令行进行手动操作。Etcd 启动参数 "--auto-compaction" 支持自动压缩 key 的历史版本，其以小时为单位。示例代码具体如下。

保留 1 个小时的历史版本：
```bash
etcd --auto-compaction-retention=1
```
用 etcdctl 命令行压缩的示例代码具体如下：

```bash
etcdctl compact 3
```

压缩之后，版本号 3 之前的 key 版本都变得不可用：

```bash
etcdctl get --rev=2 somekey
```

## 消除碎片化

压缩历史版本之后，后台数据库将会存在内部的碎片。这些碎片无法被后台存储使用，却仍占据节点的存储空间。因此消除碎片化的过程就是释放这些存储空间。压缩旧的历史版本会对后台数据库打个 ”洞”，从而导致碎片的产生。这些碎片空间对 Etcd 是可用的，但对宿主机文件系统是不可用的。

```bash
etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```

## 存储配额

Etcd 的存储配额可保证集群操作的可靠性。如果没有存储配额，那么 Etcd 的性能就会因为存储空间的持续增长而严重下降，甚至有耗完集群磁盘空间导致不可预测集群行为的风险。一旦其中一个节点的后台数据库的存储空间超出了存储配额，Etcd 就会触发集群范围的告警，并将集群置于接受读 key 和删除 key 的维护模式。只有在释放足够的空间和消除后端数据库的碎片之后，清除存储配额告警，集群才能恢复正常操作。

默认情况下，etcd 已经设置了一个适用于大部分的存储配额值，当然这个值也可以通过命令行进行配置，单个是字节。示例代码如下：

```bash
 etcd --quota-backend-bytes=$((16*1024*1024))
```

以上命令设置了一个 16MB 的存储配额。

下面将示范如何触发该配额，具体代码如下：
```bash
$ while true; do dd if=/dev/urandom bs=1024 count=1024 | ETCDCTL_API=3 etcdctl put key || break; done
...
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.0160454 s, 65.4 MB/s
OK
1024+0 records in
1024+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.0147181 s, 71.2 MB/s
Error: etcdserver: mvcc: database space exceeded
```

出现错误，此时超出了配额，下面执行命令确认是否超出了配额。
```bash
$ export ETCDCTL_API=3
$ etcdctl --write-out=table endpoint status
+----------------+------------------+---------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+---------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | 8e9e05c52164694d |  3.3.13 |   17 MB |      true |         5 |         23 |
+----------------+------------------+---------+---------+-----------+-----------+------------+
```

确认告警是否触发：
```bash
$ ETCDCTL_API=3 etcdctl alarm list
memberID:10276657743932975437 alarm:NOSPACE
```
移除超出的数据并消除后台数据的碎片之后就可以将 Etcd 的存储空间降低到配额值以下。示例代码如下.

获取当前版本号：
```bash
$ rev=$(ETCDCTL_API=3 etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
$ echo $rev
13
```
压缩旧的版本号：
```bash
$ ETCDCTL_API=3 etcdctl compact $rev
compacted revision 13
```

清除超出的存储空间的碎片：
```bash
$ ETCDCTL_API=3 etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```
消除告警：
```bash
$ ETCDCTL_API=3 etcdctl alarm disarm
memberID:10276657743932975437 alarm:NOSPACE
```
测试写操作是否恢复正常：
```bash
$ ETCDCTL_API=3 etcdctl put newkey 123
OK
```
查看存储状态：
```bash
$ ETCDCTL_API=3 etcdctl --write-out=table endpoint status
+----------------+------------------+---------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+---------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | 8e9e05c52164694d |  3.3.13 |  2.1 MB |      true |         5 |         28 |
+----------------+------------------+---------+---------+-----------+-----------+------------+
```

## 快照备份

在一个基线上为 Etcd 集群做快照能够实现 Etcd 数据的冗余备份。通过定期的为 Etcd 节点后端数据库做快照，etcd 集群就能从一个已知的良好状态的时间点进行恢复。下面的命令演示了如何将集群快照保存在 backup.db 中，然后在快速查询快照信息：

```bash
$ etcdctl snapshot save backup.db
Snapshot saved at backup.db
```
查看信息：
```bash
$ etcdctl --write-out=table snapshot status backup.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| c86911b3 |       14 |          8 |     2.1 MB |
+----------+----------+------------+------------+
```

# etcd出现如下错误  Error: etcdserver: mvcc: database space exceeded

通过执行命令压缩etcd空间并且整理空间碎片即可
```bash
#使用API3
export ETCDCTL_API=3
# 查看告警信息，告警信息一般 memberID:8630161756594109333 alarm:NOSPACE
etcdctl --endpoints=http://127.0.0.1:2379 alarm list

# 获取当前版本
rev=$(etcdctl --endpoints=http://127.0.0.1:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
# 压缩掉所有旧版本
etcdctl --endpoints=http://127.0.0.1:2379 compact $rev
# 整理多余的空间
etcdctl --endpoints=http://127.0.0.1:2379 defrag
# 取消告警信息
etcdctl --endpoints=http://127.0.0.1:2379 alarm disarm
```

说明:

压缩ETCD空间也可以减少etcd程序的内存占用量，提高etcd性能，在没有问题的时候提前进行压缩也是明智的选择

 

通过线上实践发现还是设置自动压缩更靠谱，官方同样提供数据自动压缩方式，历史数据只保留一个小时的

详细信息可以参考官方文档: https://coreos.com/etcd/docs/latest/op-guide/maintenance.html#history-compaction

```bash
etcd can be set to automatically compact the keyspace with the --auto-compaction option with a period of hours:

# keep one hour of history
$ etcd --auto-compaction-retention=1
```

根据实践发现只配置auto-compaction-retention只会做碎片整理，不会实际减少空间大小；　如果需要减少大小还是需要使用etcdctl compact 和　etcdctl defrag清理空间