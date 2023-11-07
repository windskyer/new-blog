---
title: Controller Manager简介
date: 2019-03-25 16:13:31
tags: k8s
categories: Kubernetes
---
<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# 1. Kubernetes Controller Manager简介

Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

<!-- more -->

![controller manager](http://res.cloudinary.com/dqxtn0ick/image/upload/v1510579017/article/kubernetes/core/controller-manager.png)

每个Controller通过API Server提供的接口实时监控整个集群的每个资源对象的当前状态，当发生各种故障导致系统状态发生变化时，会尝试将系统状态修复到“期望状态”。

# 2. Replication Controller

为了区分，将资源对象Replication Controller简称RC,而本文中是指Controller Manager中的Replication Controller，称为副本控制器。副本控制器的作用即保证集群中一个RC所关联的Pod副本数始终保持预设值。

1. 只有当Pod的重启策略是Always的时候（RestartPolicy=Always），副本控制器才会管理该Pod的操作（创建、销毁、重启等）。
2. RC中的Pod模板就像一个模具，模具制造出来的东西一旦离开模具，它们之间就再没关系了。一旦Pod被创建，无论模板如何变化，也不会影响到已经创建的Pod。
3. Pod可以通过修改label来脱离RC的管控，该方法可以用于将Pod从集群中迁移，数据修复等调试。
4. 删除一个RC不会影响它所创建的Pod，如果要删除Pod需要将RC的副本数属性设置为0。
5. 不要越过RC创建Pod，因为RC可以实现自动化控制Pod，提高容灾能力。

## 2.1. Replication Controller的职责

1. 确保集群中有且仅有N个Pod实例，N是RC中定义的Pod副本数量。
2. 通过调整RC中的spec.replicas属性值来实现系统扩容或缩容。
3. 通过改变RC中的Pod模板来实现系统的滚动升级。

## 2.2. Replication Controller使用场景

| 使用场景 | 说明                                       | 使用命令                   |
| ---- | ---------------------------------------- | ---------------------- |
| 重新调度 | 当发生节点故障或Pod被意外终止运行时，可以重新调度保证集群中仍然运行指定的副本数。 |                        |
| 弹性伸缩 | 通过手动或自动扩容代理修复副本控制器的spec.replicas属性，可以实现弹性伸缩。 | kubectl scale          |
| 滚动更新 | 创建一个新的RC文件，通过kubectl 命令或API执行，则会新增一个新的副本同时删除旧的副本，当旧副本为0时，删除旧的RC。 | kubectl rolling-update |

滚动升级，具体可参考kubectl rolling-update --help,官方文档：[https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/)

# 3. Node Controller

kubelet在启动时会通过API Server注册自身的节点信息，并定时向API Server汇报状态信息，API Server接收到信息后将信息更新到etcd中。

Node Controller通过API Server实时获取Node的相关信息，实现管理和监控集群中的各个Node节点的相关控制功能。流程如下

![Node Controller](https://res.cloudinary.com/dqxtn0ick/image/upload/v1510579017/article/kubernetes/core/NodeController.png)

1、Controller Manager在启动时如果设置了--cluster-cidr参数，那么为每个没有设置Spec.PodCIDR的Node节点生成一个CIDR地址，并用该CIDR地址设置节点的Spec.PodCIDR属性，防止不同的节点的CIDR地址发生冲突。

2、具体流程见以上流程图。

3、逐个读取节点信息，如果节点状态变成非“就绪”状态，则将节点加入待删除队列，否则将节点从该队列删除。

# 4. ResourceQuota Controller

资源配额管理确保指定的资源对象在任何时候都不会超量占用系统物理资源。

支持三个层次的资源配置管理：

1）容器级别：对CPU和Memory进行限制

2）Pod级别：对一个Pod内所有容器的可用资源进行限制

3）Namespace级别：包括

- Pod数量
- Replication Controller数量
- Service数量
- ResourceQuota数量
- Secret数量
- 可持有的PV（Persistent Volume）数量

说明：

1. k8s配额管理是通过Admission Control（准入控制）来控制的；
2. Admission Control提供两种配额约束方式：LimitRanger和ResourceQuota；
3. LimitRanger作用于Pod和Container；
4. ResourceQuota作用于Namespace上，限定一个Namespace里的各类资源的使用总额。

**ResourceQuota Controller流程图**：

![ResourceQuota Controller](https://res.cloudinary.com/dqxtn0ick/image/upload/v1510579017/article/kubernetes/core/ResourceQuotaController.png)

# 5. Namespace Controller

用户通过API Server可以创建新的Namespace并保存在etcd中，Namespace Controller定时通过API Server读取这些Namespace信息。

如果Namespace被API标记为优雅删除（即设置删除期限，DeletionTimestamp）,则将该Namespace状态设置为“Terminating”,并保存到etcd中。同时Namespace Controller删除该Namespace下的ServiceAccount、RC、Pod等资源对象。

# 6. Endpoint Controller

**Service、Endpoint、Pod的关系：**

![Endpoint Controller](https://res.cloudinary.com/dqxtn0ick/image/upload/v1510579017/article/kubernetes/core/EndpointController.png)

Endpoints表示了一个Service对应的所有Pod副本的访问地址，而Endpoints Controller负责生成和维护所有Endpoints对象的控制器。它负责监听Service和对应的Pod副本的变化。

1. 如果监测到Service被删除，则删除和该Service同名的Endpoints对象；
2. 如果监测到新的Service被创建或修改，则根据该Service信息获得相关的Pod列表，然后创建或更新Service对应的Endpoints对象。
3. 如果监测到Pod的事件，则更新它对应的Service的Endpoints对象。

kube-proxy进程获取每个Service的Endpoints，实现Service的负载均衡功能。

# 7. Service Controller

Service Controller是属于kubernetes集群与外部的云平台之间的一个接口控制器。Service Controller监听Service变化，如果是一个LoadBalancer类型的Service，则确保外部的云平台上对该Service对应的LoadBalancer实例被相应地创建、删除及更新路由转发表。 


# Controller Manager

Controller Manager 由 kube-controller-manager 和 cloud-controller-manager 组成，是 Kubernetes 的大脑，它通过 apiserver 监控整个集群的状态，并确保集群处于预期的工作状态。

![](images/post-ccm-arch.png)

kube-controller-manager 由一系列的控制器组成

- Replication Controller
- Node Controller
- CronJob Controller
- Daemon Controller
- Deployment Controller
- Endpoint Controller
- Garbage Collector
- Namespace Controller
- Job Controller
- Pod AutoScaler
- RelicaSet
- Service Controller
- ServiceAccount Controller
- StatefulSet Controller
- Volume Controller
- Resource quota Controller

cloud-controller-manager 在 Kubernetes 启用 Cloud Provider 的时候才需要，用来配合云服务提供商的控制，也包括一系列的控制器，如

- Node Controller
- Route Controller
- Service Controller

从 v1.6 开始，cloud provider 已经经历了几次重大重构，以便在不修改 Kubernetes 核心代码的同时构建自定义的云服务商支持。参考 [这里](../plugins/cloud-provider.md) 查看如何为云提供商构建新的 Cloud Provider。

## Metrics

Controller manager metrics 提供了控制器内部逻辑的性能度量，如 Go 语言运行时度量、etcd 请求延时、云服务商 API 请求延时、云存储请求延时等。Controller manager metrics 默认监听在 `kube-controller-manager` 的 10252 端口，提供 Prometheus 格式的性能度量数据，可以通过 `http://localhost:10252/metrics` 来访问。

```
$ curl http://localhost:10252/metrics
...
# HELP etcd_request_cache_add_latencies_summary Latency in microseconds of adding an object to etcd cache
# TYPE etcd_request_cache_add_latencies_summary summary
etcd_request_cache_add_latencies_summary{quantile="0.5"} NaN
etcd_request_cache_add_latencies_summary{quantile="0.9"} NaN
etcd_request_cache_add_latencies_summary{quantile="0.99"} NaN
etcd_request_cache_add_latencies_summary_sum 0
etcd_request_cache_add_latencies_summary_count 0
# HELP etcd_request_cache_get_latencies_summary Latency in microseconds of getting an object from etcd cache
# TYPE etcd_request_cache_get_latencies_summary summary
etcd_request_cache_get_latencies_summary{quantile="0.5"} NaN
etcd_request_cache_get_latencies_summary{quantile="0.9"} NaN
etcd_request_cache_get_latencies_summary{quantile="0.99"} NaN
etcd_request_cache_get_latencies_summary_sum 0
etcd_request_cache_get_latencies_summary_count 0
...
```

## kube-controller-manager 启动示例

```sh
kube-controller-manager \
  --enable-dynamic-provisioning=true \
  --feature-gates=AllAlpha=true \
  --horizontal-pod-autoscaler-sync-period=10s \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --node-monitor-grace-period=10s \
  --address=127.0.0.1 \
  --leader-elect=true \
  --kubeconfig=/etc/kubernetes/controller-manager.conf \
  --cluster-signing-key-file=/etc/kubernetes/pki/ca.key \
  --use-service-account-credentials=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --root-ca-file=/etc/kubernetes/pki/ca.crt \
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.244.0.0/16 \
  --node-cidr-mask-size=24
```

## 控制器

### kube-controller-manager

kube-controller-manager 由一系列的控制器组成，这些控制器可以划分为三组

1. 必须启动的控制器
   - EndpointController
   - ReplicationController：
   - PodGCController
   - ResourceQuotaController
   - NamespaceController
   - ServiceAccountController
   - GarbageCollectorController
   - DaemonSetController
   - JobController
   - DeploymentController
   - ReplicaSetController
   - HPAController
   - DisruptionController
   - StatefulSetController
   - CronJobController
   - CSRSigningController
   - CSRApprovingController
   - TTLController
2. 默认启动的可选控制器，可通过选项设置是否开启
   - TokenController
   - NodeController
   - ServiceController
   - RouteController
   - PVBinderController
   - AttachDetachController
3. 默认禁止的可选控制器，可通过选项设置是否开启
   - BootstrapSignerController
   - TokenCleanerController

### cloud-controller-manager

cloud-controller-manager 在 Kubernetes 启用 Cloud Provider 的时候才需要，用来配合云服务提供商的控制，也包括一系列的控制器

- CloudNodeController
- RouteController
- ServiceController

## 高可用

在启动时设置 `--leader-elect=true` 后，controller manager 会使用多节点选主的方式选择主节点。只有主节点才会调用 `StartControllers()` 启动所有控制器，而其他从节点则仅执行选主算法。

多节点选主的实现方法见 [leaderelection.go](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go)。它实现了两种资源锁（Endpoint 或 ConfigMap，kube-controller-manager 和 cloud-controller-manager 都使用 Endpoint 锁），通过更新资源的 Annotation（`control-plane.alpha.kubernetes.io/leader`），来确定主从关系。

## 高性能

从 Kubernetes 1.7 开始，所有需要监控资源变化情况的调用均推荐使用 [Informer](https://github.com/kubernetes/client-go/blob/master/tools/cache/shared_informer.go)。Informer 提供了基于事件通知的只读缓存机制，可以注册资源变化的回调函数，并可以极大减少 API 的调用。

Informer 的使用方法可以参考 [这里](https://github.com/feiskyer/kubernetes-handbook/tree/master/examples/client/informer)。

## Node Eviction

Node 控制器在节点异常后，会按照默认的速率（`--node-eviction-rate=0.1`，即每10秒一个节点的速率）进行 Node 的驱逐。Node 控制器按照 Zone 将节点划分为不同的组，再跟进 Zone 的状态进行速率调整：

- Normal：所有节点都 Ready，默认速率驱逐。
- PartialDisruption：即超过33% 的节点 NotReady 的状态。当异常节点比例大于 `--unhealthy-zone-threshold=0.55` 时开始减慢速率：
  - 小集群（即节点数量小于 `--large-cluster-size-threshold=50`）：停止驱逐
  - 大集群，减慢速率为 `--secondary-node-eviction-rate=0.01`
- FullDisruption：所有节点都 NotReady，返回使用默认速率驱逐。但当所有 Zone 都处在 FullDisruption 时，停止驱逐。

#什么是 DaemonSet？

DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

运行集群存储 daemon，例如在每个 Node 上运行 glusterd、ceph。
在每个 Node 上运行日志收集 daemon，例如fluentd、logstash。
在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter、collectd、Datadog 代理、New Relic 代理，或 Ganglia gmond。
一个简单的用法是，在所有的 Node 上都存在一个 DaemonSet，将被作为每种类型的 daemon 使用。 一个稍微复杂的用法可能是，对单独的每种类型的 daemon 使用多个 DaemonSet，但具有不同的标志，和/或对不同硬件类型具有不同的内存、CPU要求。

# 编写 DaemonSet Spec

## 必需字段

和其它所有 Kubernetes 配置一样，DaemonSet 需要 apiVersion、kind 和 metadata字段。有关配置文件的通用信息，详见文档 deploying applications、配置容器 和 资源管理 。

DaemonSet 也需要一个 .spec 配置段。

## Pod 模板

.spec 唯一必需的字段是 .spec.template。

.spec.template 是一个 Pod 模板。 它与 Pod 具有相同的 schema，除了它是嵌套的，而且不具有 apiVersion 或 kind 字段。

Pod 除了必须字段外，在 DaemonSet 中的 Pod 模板必须指定合理的标签（查看 pod selector）。

在 DaemonSet 中的 Pod 模板必需具有一个值为 Always 的 RestartPolicy，或者未指定它的值，默认是 Always。

Pod Selector
.spec.selector 字段表示 Pod Selector，它与 Job 或其它资源的 .sper.selector 的原理是相同的。

spec.selector 表示一个对象，它由如下两个字段组成：

matchLabels - 与 ReplicationController 的 .spec.selector 的原理相同。
matchExpressions - 允许构建更加复杂的 Selector，可以通过指定 key、value 列表，以及与 key 和 value 列表的相关的操作符。
当上述两个字段都指定时，结果表示的是 AND 关系。

如果指定了 .spec.selector，必须与 .spec.template.metadata.labels 相匹配。如果没有指定，它们默认是等价的。如果与它们配置的不匹配，则会被 API 拒绝。

如果 Pod 的 label 与 selector 匹配，或者直接基于其它的 DaemonSet、或者 Controller（例如 ReplicationController），也不可以创建任何 Pod。 否则 DaemonSet Controller 将认为那些 Pod 是它创建的。Kubernetes 不会阻止这样做。一个场景是，可能希望在一个具有不同值的、用来测试用的 Node 上手动创建 Pod。

## 仅在相同的 Node 上运行 Pod

如果指定了 .spec.template.spec.nodeSelector，DaemonSet Controller 将在能够匹配上 Node Selector 的 Node 上创建 Pod。 类似这种情况，可以指定 .spec.template.spec.affinity，然后 DaemonSet Controller 将在能够匹配上 Node Affinity 的 Node 上创建 Pod。 如果根本就没有指定，则 DaemonSet Controller 将在所有 Node 上创建 Pod。

## 如果调度 Daemon Pod

正常情况下，Pod 运行在哪个机器上是由 Kubernetes 调度器进行选择的。然而，由 Daemon Controller 创建的 Pod 已经确定了在哪个机器上（Pod 创建时指定了 .spec.nodeName），因此：

DaemonSet Controller 并不关心一个 Node 的 unschedulable 字段。
DaemonSet Controller 可以创建 Pod，即使调度器还没有被启动，这对集群启动是非常有帮助的。
Daemon Pod 关心 Taint 和 Toleration，它们会为没有指定 tolerationSeconds 的 node.alpha.kubernetes.io/notReady 和 node.alpha.kubernetes.io/unreachable 的 Taint，而创建具有 NoExecute 的 Toleration。这确保了当 alpha 特性的 TaintBasedEvictions 被启用，当 Node 出现故障，比如网络分区，这时它们将不会被清除掉（当 TaintBasedEvictions 特性没有启用，在这些场景下也不会被清除，但会因为 NodeController 的硬编码行为而被清除，Toleration 是不会的）。

## 与 Daemon Pod 通信

与 DaemonSet 中的 Pod 进行通信，几种可能的模式如下：

Push：配置 DaemonSet 中的 Pod 向其它 Service 发送更新，例如统计数据库。它们没有客户端。
NodeIP 和已知端口：DaemonSet 中的 Pod 可以使用 hostPort，从而可以通过 Node IP 访问到 Pod。客户端能通过某种方法知道 Node IP 列表，并且基于此也可以知道端口。
DNS：创建具有相同 Pod Selector 的 Headless Service，然后通过使用 endpoints 资源或从 DNS 检索到多个 A 记录来发现 DaemonSet。
Service：创建具有相同 Pod Selector 的 Service，并使用该 Service 访问到某个随机 Node 上的 daemon。（没有办法访问到特定 Node）

## 更新 DaemonSet

如果修改了 Node Label，DaemonSet 将立刻向新匹配上的 Node 添加 Pod，同时删除新近无法匹配上的 Node 上的 Pod。

可以修改 DaemonSet 创建的 Pod。然而，不允许对 Pod 的所有字段进行更新。当下次 Node（即使具有相同的名称）被创建时，DaemonSet Controller 还会使用最初的模板。

可以删除一个 DaemonSet。如果使用 kubectl 并指定 --cascade=false 选项，则 Pod 将被保留在 Node 上。然后可以创建具有不同模板的新 DaemonSet。具有不同模板的新 DaemonSet 将鞥能够通过 Label 匹配识别所有已经存在的 Pod。它不会修改或删除它们，即使是错误匹配了 Pod 模板。通过删除 Pod 或者 删除 Node，可以强制创建新的 Pod。

在 Kubernetes 1.6 或以后版本，可以在 DaemonSet 上 执行滚动升级。

未来的 Kubernetes 版本将支持 Node 的可控更新。

## DaemonSet 的可替代选择

### init 脚本

很可能通过直接在一个 Node 上启动 daemon 进程（例如，使用 init、upstartd、或 systemd）。这非常好，然而基于 DaemonSet 来运行这些进程有如下一些好处：

像对待应用程序一样，具备为 daemon 提供监控和管理日志的能力。
为 daemon 和应用城西使用相同的配置语言和工具（如 Pod 模板、kubectl）。
Kubernetes 未来版本可能会支持对 DaemonSet 创建 Pod 与 Node升级工作流进行集成。
在资源受限的容器中运行 daemon，能够增加 daemon 和应用容器的隔离性。然而这也实现了在容器中运行 daemon，但却不能在 Pod 中运行（例如，直接基于 Docker 启动）。
裸 Pod
可能要直接创建 Pod，同时指定其运行在特定的 Node 上。 然而，DaemonSet 替换了由于任何原因被删除或终止的 Pod，例如 Node 失败、例行节点维护，比如内和升级。由于这个原因，我们应该使用 DaemonSet 而不是单独创建 Pod。

### 静态 Pod

很可能，通过在一个指定目录下编写文件来创建 Pod，该目录受 Kubelet 所监视。这些 Pod 被称为 静态 Pod。 不像 DaemonSet，静态 Pod 不受 kubectl 和 其它 Kubernetes API 客户端管理。静态 Pod 不依赖于 apiserver，这使得它们在集群启动的情况下非常有用。 而且，未来静态 Pod 可能会被废弃掉。

### Replication Controller

DaemonSet 与 Replication Controller 非常类似，它们都能创建 Pod，这些 Pod 都具有不期望被终止的进程（例如，Web 服务器、存储服务器）。 为无状态的 Service 使用 Replication Controller，像 frontend，实现对副本的数量进行扩缩容、平滑升级，比之于精确控制 Pod 运行在某个主机上要重要得多。需要 Pod 副本总是运行在全部或特定主机上，并需要先于其他 Pod 启动，当这被认为非常重要时，应该使用 Daemon Controller。