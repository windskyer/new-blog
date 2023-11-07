---
title: 自动扩缩集群 DNS 服务
categories: Kubernetes
sage: false
date: 2021-03-12 11:37:08
tags: dns
---

# 简介
DNSAutoscaler 是 DNS 自动水平伸缩组件，可通过一个 deployment 获取集群的节点数和核数，根据预设的伸缩策略，自动水平伸缩 DNS 的副本数。目前的伸缩模式分为两种，分别是 Linear 线性模式 和 Ladder 阶梯模式。

## 组件介绍 

我们创建一个 Deployment。Deployment 中的 Pod 运行一个基于 cluster-proportional-autoscaler 镜像的容器。

创建文件 dns-horizontal-autoscaler.yaml，内容如下所示：
<!-- more -->
```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: cluster-proportional-autoscaler
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-proportional-autoscaler
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources: ["replicationcontrollers/scale"]
    verbs: ["get", "update"]
  - apiGroups: ["extensions","apps"]
    resources: ["deployments/scale", "replicasets/scale"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-proportional-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-proportional-autoscaler
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-proportional-autoscaler
  apiGroup: rbac.authorization.k8s.io
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: dns-autoscaler
  namespace: kube-system
data:
  linear: |-
    {
      "coresPerReplica": 256,
      "nodesPerReplica": 3,
      "min": 1,
      "max": 100,
      "preventSinglePointFailure": true,
      "includeUnschedulableNodes": true
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-autoscaler
  namespace: kube-system
  labels:
    k8s-app: dns-autoscaler
spec:
  selector:
    matchLabels:
      k8s-app: dns-autoscaler
  template:
    metadata:
      labels:
        k8s-app: dns-autoscaler
    spec:
      serviceAccountName: cluster-proportional-autoscaler
      containers:
      - name: autoscaler
        image: hub.kce.ksyun.com/ksyun/cluster-proportional-autoscaler:1.8.5
        resources:
          requests:
            cpu: 20m
            memory: 10Mi
        command:
        - /cluster-proportional-autoscaler
        - --namespace=kube-system
        - --configmap=dns-autoscaler
        - --target=deployment/coredns
        # When cluster is using large nodes(with more cores), "coresPerReplica" should dominate.
        # If using small nodes, "nodesPerReplica" should dominate.
        # - --default-params={"linear":{"coresPerReplica":256,"nodesPerReplica":16,"min":1}}
        - --logtostderr=true
        - --v=2
```
```sh
kubectl apply -f dns-horizontal-autoscaler.yaml
```
## 配置介绍

#### Linear Mode
ConfigMap 配置示例如下：
```yaml
data:
  linear: |-
    {
      "coresPerReplica": 2,
      "nodesPerReplica": 1,
      "min": 1,
      "max": 100,
      "preventSinglePointFailure": true,
      "includeUnschedulableNodes": true
    }
```

目标副本计算公式：
replicas = max( ceil( cores * 1/coresPerReplica ) , ceil( nodes * 1/nodesPerReplica ) )
replicas = min(replicas, max)
replicas = max(replicas, min)

>1. 当 preventSinglePointFailure 设置为 true 时，如果有多个节点，则控制器确保至少有 2 个副本
>2. 当 includeUnschedulableNodes 设置为 true 时，副本将根据节点总数进行扩展。否则，副本将仅根据可调度节点的数量进行扩展（即，排除了封锁和排空节点。）

例如，给定一个集群有 4 个节点和 13 个核。通过以上参数，每个副本可以处理 1 个节点。所以我们需要 4 / 1 = 4 个副本来处理所有 4 个节点。每个副本可以处理 2 个核。我们需要 ceil(13 / 2) = 7 个副本来处理所有 13 个核。控制器将选择较大的一个，这里是 7，作为结果。

可以省略 coresPerReplica 或 nodesPerReplica 之一。 min、max、preventSinglePointFailure 和 includeUnscheduleableNodes 都是可选的。如果未设置，min 将默认为 1，preventSinglePointFailure 将默认为 false，includeUnschedulableNodes 将默认为 false。

注意：
>1. coresPerReplica 和 nodesPerReplica 都是浮动的。 
>2. 当 min 小于 1 时，最低副本将设置为 1。

#### Ladder Mode
ConfigMap 配置示例如下：
```yaml
data:
  ladder: |-
    {
      "coresToReplicas":
      [
        [ 1, 1 ],
        [ 64, 3 ],
        [ 512, 5 ],
        [ 1024, 7 ],
        [ 2048, 10 ],
        [ 4096, 15 ]
      ],
      "nodesToReplicas":
      [
        [ 1, 1 ],
        [ 2, 2 ]
      ]
    }
```
目标副本计算：
假设 100nodes/400cores 的集群中，按上述配置，nodesToReplicas 取2（100>2)，coresToReplicas 取3（64<400<512），二者取较大值3，最终 replica 为3。