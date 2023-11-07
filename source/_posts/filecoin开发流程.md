---
title: filecoin介绍
categories: coin
sage: false
date: 2022-09-16 14:35:49
tags: coin
---

# BingFile 管理系统

## filecoin 项目介绍

### IPFS、Filecoin和lotus的介绍及其关系

#### IPFS
IPFS（全名为InterPlanetary File System，中文名翻译成星际文件系统或者分布式文件系统），是一种硬盘共享的互联网底层协议。它的创始人是Juan Benet（胡安·贝内特）， 于2015年5月上线。

IPFS创建的目的：HTTP是中心化，每次获取文件都需要从中心化的服务器下载，所以在某种层度上会出现速度慢、效率低和成本高。为了解决HTTP协议的不足，IPFS使用P2P模式来下载所需文件，将文件分割成小块滨崎可以同时在多个服务器下载。因此IPFS的目的是取代HTTP协议，使互联网更加美好。

ipfs官网：https://ipfs.io/
ifps开源代码：https://github.com/ipfs/

#### Filecoin
Filecoin是一个项目，它通过IPFS技术和区块链技术来建立的一个去中心化的存储市场。
Filecoin 是一种将文件存储在互联网上的点对点网络，具有内置的经济激励措施以确保文件随着时间的推移可靠地存储。
Filecoin包括区块链和FIL。矿工可以通过存储文件或区块链上出块来获取FIL。Filecoin的区块链上记录着不可篡改的信息。

Filecoin的主网于2020年10月15日上市，不是上线哦。因为它原本就早就上线了，只是一直没上市。原本叫测试网，上市那天直接改成主网，区块链上的信息没有清除，还保留着。所以，如果你是2020年10月15日之前加入Filecoin，可以免费获取FIL。

Filecoin官网：https://filecoin.io/zh-cn/
官方对Filecoin的介绍：https://docs.filecoin.io/about-filecoin/what-is-filecoin/
Filecoin官方区块链浏览器：https://stats.filecoin.io/

有些第三方的Filecoin区块链浏览器写得比官方好，如
https://filfox.info/zh
https://filscan.io/

#### lotus
Filecoin 的实现包括go-filecoin，lotus等。目前成功实现的是lotus。
lotus是Filecoin 协议规范的最简且具有实验性的实现，用 Go 语言编写。

如果你想加入Filecoin的区块链，必须通过lotus。

lotus开源代码：https://github.com/filecoin-project/lotus
lotus官方文档：https://docs.filecoin.io/get-started/lotus/
（之前的文档是https://lotu.sh/，不过现在已经弃用了）

### lotus 工具介绍

>1. lotus: 主要程序
>2. lotus-miner: 矿工程序
>3. lotus-worker: 矿工子程序
>4. lotus-seed: 初始化扇区工具

lotus简介：
filecoin项目主程序，下载证明参数和启动守护进程。


lotus-worker简介：
Worker 并不是必须的，miner 自带就有一个 worker， worker 能做的， miner 都能做，worker 的唯一作用只是为了分担 miner 的任务罢了。 对于迷你旷工（比如只有一台机器的），就完全不需要启动 worker 了，有一个 miner 就完全足够了，但对于小集群，或者大集群，worker 就是必须的了，因此，这一节讲述如何使用 worker。

lotus-seed简介：
初始化扇区， 用作本地自己创建的网络需要启动 新建创世节点。跑主网的时候不需要你来启动创世节点，因为创世节点已经由官方启动了。



### 下载证明参数
```sh
# 2KiB 表示我们要下载的证明参数是 2KiB 大小的扇区所需要的证明参数
# 主网上使用的是 32GiB 的证明参数，或者是 64GiB 的证明参数
lotus fetch-params 2KiB
```

下载证明参数的过程可能会比较慢，因为证明参数放在国外的服务器，如果你有代理，可以挂上代理再下载。 此外，也可以从 JDCloud 上面下载，你需要做的事情就是在执行上述命令之前设置一下 JDCloud 的代理：

```sh
# 这个地址下载的参数有问题，请不要再设置这个环境变量，直接下载即可
# 最好挂上代理，因为这个证明参数是放在国外服务器上，国内下载相对较慢，晚上下载会快些
export IPFS_GATEWAY=https://proof-parameters.s3.cn-south-1.jdcloud-oss.com/ipfs/
```
## filecoin 监控系统

### 磁盘采集选项
计算方法：先读取/proc/mounts拿到所有挂载点，然后通过syscall.Statfs_t拿到blocks和inode的使用情况。每个metric都会附加一组tag描述，类似mount=$mount,fstype=$fstype，其中$mount是挂载点，比如/home，$fstype是文件系统，比如ext4。
>1. df.bytes.free：磁盘可用量，int64
>2. df.bytes.free.percent：磁盘可用量占总量的百分比，float64，比如32.1
>3. df.bytes.total：磁盘总大小，int64
>4. df.bytes.used：磁盘已用大小，int64
>5. df.bytes.used.percent：磁盘已用大小占总量的百分比，float64
>6. df.inodes.total：inode总数，int64
>7. df.inodes.free：可用inode数目，int64
>8. df.inodes.free.percent：可用inode占比，float64
>9. df.inodes.used：已用的inode数据，int64
>10. df.inodes.used.percent：已用inode占比，float64
#### megacli工具输出
使用 megacli 工具读取 RAID 相关信息，每个metric都会附件一组tag描述，用来标明所属PD或者 VD，PD格式为PD=Enclosure_ID:SLOT_ID，比如PD=32:0表明第一块磁盘 ，VD=0 表明第一个逻辑磁盘。

>1. sys.disk.lsiraid.pd.Media_Error_Count：这个及以下三个指标目前仅作为数据收集，不一定意味磁盘损坏（只是表示损坏概率变大）
>2. sys.disk.lsiraid.pd.Other_Error_Count
>3. sys.disk.lsiraid.pd.Predictive_Failure_Count
>4. sys.disk.lsiraid.pd.Drive_Temperature
>5. sys.disk.lsiraid.pd.Firmware_state：如果值不为0，则此物理磁盘出现问题
>6. sys.disk.lsiraid.vd.cache_policy：如果值不为0，表示此逻辑磁盘缓存策略和设置不符
>7. sys.disk.lsiraid.vd.state： 如果值不为0，表示此逻辑磁盘出现问题

#### smart工具输出
使用 smartctl 工具读取磁盘 SMART 信息，目前所有指标仅作为数据收集，不一定意味磁盘损坏（只是表示概率变大），每个metric都会有一组tag描述，表明盘符，例如device=/dev/sda。

>1. sys.disk.smart.Reallocated_Sector_Ct
>2. sys.disk.smart.Spin_Retry_Count
>3. sys.disk.smart.Reallocated_Event_Count
>4. sys.disk.smart.Current_Pending_Sector
>5. sys.disk.smart.Offline_Uncorrectable
>6. sys.disk.smart.Temperature_Celsius

#### 分区读写监控
测试所有已挂载分区是否可读写，每个metric都会有一组tag描述，表示挂载点，比如mount=/home

>1. sys.disk.rw： 如果值不为0，表明此分区读写出现问题

#### IO相关采集项
计算方法：每秒采集一次/proc/diskstats，计算差值，都是计数器类型的。每个metric都会有一组tag描述，形如device=$device，用来表示具体的设备，比如sda1、sdb。用户可以参考iostat的帮助文档来理解具体的metric含义。

>1. disk.io.ios_in_progress：当前正在运行的实际 I/O 请求数。
>2. disk.io.msec_read：所有读取花费的总毫秒数。
>3. disk.io.msec_total：ios_in_progress >= 1 的时间量。
>4. disk.io.msec_weighted_total：衡量最近的 I/O 完成时间和块。
>5. disk.io.msec_write：所有写入花费的总毫秒数。
>6. disk.io.read_merged：相邻的读取请求合并在一个请求中。
>7. disk.io.read_requests：Total number of reads completed successfully.
>8. disk.io.read_sectors：Total number of sectors read successfully.
>9. disk.io.write_merged：Adjacent write requests merged in a single req.
>10. disk.io.write_requests：total number of writes completed successfully.
>11. disk.io.write_sectors：total number of sectors written successfully.
>12. disk.io.read_bytes：单位是byte的数字
>13. disk.io.write_bytes：单位是byte的数字
>14. disk.io.avgrq_sz：下面几个值就是iostat -x 1看到的值
>15. disk.io.avgqu-sz
>16. disk.io.await
>17. disk.io.svctm
>18. disk.io.util：是个百分数，比如56.43，表示56.43%

#### 网络相关采集项
计算方法：读取/proc/net/dev的内容，每个metric都附加有一组tag，形如iface=$iface，标明具体那个interface，比如eth0。metric中带有in的表示流入情况，out表示流出情况，total是总量in+out，支持的metric如下：

>1. net.if.in.bytes
>2. net.if.in.compressed
>3. net.if.in.dropped
>4. net.if.in.errors
>5. net.if.in.fifo.errs
>6. net.if.in.frame.errs
>7. net.if.in.multicast
>8. net.if.in.packets
>9. net.if.out.bytes
>10. net.if.out.carrier.errs
>11. net.if.out.collisions
>12. net.if.out.compressed
>13. net.if.out.dropped
>14. net.if.out.errors
>15. net.if.out.fifo.errs
>16. net.if.out.packets
>17. net.if.total.bytes
>18. net.if.total.dropped
>19. net.if.total.errors
>20. net.if.total.packets

#### 进程资源监控

>1. process.cpu.all：进程和它的子进程使用的sys+user的cpu，单位是jiffies
>2. process.cpu.sys：进程和它的子进程使用的sys cpu，单位是jiffies
>3. process.cpu.user：进程和它的子进程使用的user cpu，单位是jiffies
>4. process.swap：进程和它的子进程使用的swap，单位是page
>5. process.fd：进程使用的文件描述符个数
>6. process.mem：进程占用内存，单位byte

## filecoin 后台系统(bingFile)

### 功能模块分解

#### 算力管理
支持市场主流X86和ARM CPU指令集服务器。
支持主流鲲鹏，飞腾等国产化服务器。

#### 任务管理
支持矿机节点安装，重启，下架，更新，登入等操作。

#### 用户管理
支持对系统用户进行管理，包含新增、查询、编辑、禁用启用等，可以根据业务需要配置用户对系统的操作权限。

#### 数据总览(大屏展示)
总览大屏，是系统的总体运行情况和资源使用情况信息展示界面，支持运行任务统计和信息展示、处理数据量统计展示、集群节点统计信息展示、算力单元信息统计展示、负载信息统计展示等。

#### 资源监控
支持对系统资源情况的实时监测，包含CPU、内存、GPU、DISK等资源，支持对钱包，fil币池展示，支持对各类资源已分配和已使用情况的统计展示，支持对集群节点数量、状态等信息的查看。
