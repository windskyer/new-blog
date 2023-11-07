---
title: k8s亲和性
date: 2019-03-01 12:06:13
tags: scheduler
categories: Kubernetes
---

# Kubernetes中的亲和性

现实中应用的运行对于kubernetes在亲和性上提出了一些要求，可以归类到以下几个方面：
>1.Pod固定调度到某些节点之上
>2.Pod不会调度到某些节点之上
>3.Pod的多副本调度到相同的节点之上
>4.Pod的多副本调度到不同的节点之上

<!-- more -->

## k8s 亲和性包含

```go
// Affinity is a group of affinity scheduling rules.
type Affinity struct {
	// Describes node affinity scheduling rules for the pod.
	// +optional
	NodeAffinity *NodeAffinity `json:"nodeAffinity,omitempty" protobuf:"bytes,1,opt,name=nodeAffinity"`
	// Describes pod affinity scheduling rules (e.g. co-locate this pod in the same node, zone, etc. as some other pod(s)).
	// +optional
	PodAffinity *PodAffinity `json:"podAffinity,omitempty" protobuf:"bytes,2,opt,name=podAffinity"`
	// Describes pod anti-affinity scheduling rules (e.g. avoid putting this pod in the same node, zone, etc. as some other pod(s)).
	// +optional
	PodAntiAffinity *PodAntiAffinity `json:"podAntiAffinity,omitempty" protobuf:"bytes,3,opt,name=podAntiAffinity"`
}
```

>- NodeAffinity 相对于nodeSelector机制更加的灵活和丰富, 丰富nodeSelector 机制
>- PodAffinity 设置pod 的亲和性
>- PodAntiAffinity 设置pod 的反亲和性

## NodeAffinity 实例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 10
        preference:
          matchExpressions:
          - key: key1
            operator: In
            values:
            - value1
      - weight: 2
        preference:
          matchExpressions:
          - key: key2
            operator: In
            values:
            - value2
  containers:
  - name: with-node-affinity
    image: hub.kce.ksyun.com/ksyun/pause-amd64:3.0
```

>- 表达的语法：支持In,NotIn,Exists,DoesNotExist,Gt,Lt．
>- 支持soft(preference)和hard(requirement),hard表示pod sheduler到某个node上，则必须满足亲和性设置．soft表示scheduler的时候，无法满足节点的时候，会选择非nodeSelector匹配的节点．
>- requiredDuringSchedulingIgnoredDuringExecution 必须满足条件
>- preferredDuringSchedulingIgnoredDuringExecution 非必须满足条件
>- 如上: node label 中必须有kubernetes.io/e2e-az-name=e2e-az1 或者 kubernetes.io/e2e-az-name=e2e-az2
>- 如上: node label 中有key1 的优先权大于 key2 的节点

补充：

>1. nodeSelectorTerms 选项中可以添加 多个 matchExpressions，多个matchFields
>2. preferredDuringSchedulingIgnoredDuringExecution 可以写多个 weight项
**注意：测试v1.10.5， v1.12.3版本nodeSelectorTerms 只需要满足nodeSelectorTerms其中一个选项就可以调度**

## PodAffinity 实例

>- 这个特性是Kubernetes 1.4后增加的，允许用户通过已经运行的Pod上的标签来决定调度策略，用文字描述就是“如果Node X上运行了一个或多个满足Y条件的Pod，那么这个Pod在Node应该运行在Pod X”，因为Node没有命名空间，Pod有命名空间，这样就允许管理员在配置的时候指定这个亲和性策略适用于哪个命名空间，可以通过topologyKey来指定。topology是一个范围的概念，可以是一个Node、一个机柜、一个机房或者是一个区域（如北美、亚洲）等，实际上对应的还是Node上的标签。
**有两种类型**
>- requiredDuringSchedulingIgnoredDuringExecution，刚性要求，必须精确匹配
>- preferredDuringSchedulingIgnoredDuringExecution，软性要求
>- 类似上面node的亲和策略类似，requiredDuringSchedulingIgnoredDuringExecution亲和性可以用于约束不同服务的pod在同一个topology domain的Nod上preferredDuringSchedulingIgnoredDuringExecution反亲和性可以将服务的pod分散到不同的topology domain的Node上．
**标签支持**
>- 标签的判断操作支持In、NotIn、Exists、DoesNotExist。
**原则上topologyKey可以是节点的合法标签，但是有一些约束：**
>- kubernetes.io/hostname　　＃Node
>- failure-domain.beta.kubernetes.io/zone　＃Zone
>- failure-domain.beta.kubernetes.io/region #Region
>- 可以设置node上的label的值来表示node的name,zone,region等信息，pod的规则中指定topologykey的值表示指定topology范围内的node上运行的pod满足指定规则

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: hub.kce.ksyun.com/ksyun/pause-amd64:3.0
```

**例子中指定了pod的亲和性和反亲和性， preferredDuringSchedulingIgnoredDuringExecution指定的规则是pod将会调度到的node尽量会满足如下条件：**

>- node上具有failure-domain.beta.kubernetes.io/zone
>- 并且具有相同failure-domain.beta.kubernetes.io/zone的值的node上运行有一个pod,它符合label为securtity=S1.
>- preferredDuringSchedulingIgnoredDuringExecution规则表示将尽量不会调度到node上运行有security=S2的pod．
>- 如果这里我们将topologyKey＝failure-domain.beta.kubernetes.io/zone，那么pod将尽量不会调度到node满足的条件是：node上具有failure-domain.beta.kubernetes.io/zone相同的ｖalue,并且这些相同zone下的node上运行有security=S2的pod.**

## 总结

>1. Pod间的亲和性策略要求可观的计算量可能显著降低集群的性能，不建议在超过100台节点的范围内使用。
>2. Pod间的反亲和策略要求所有的Node都有一致的标签，例如集群中所有节点都应有匹配topologyKey的标签，如果一些节点缺失这些标签可能导致异常行为。

## 常用的场景

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

**注解：上面的例子中，创建了一个具有三个实例的部署，采用了Pod间的反亲和策略，限制创建的实例的时候，如果节点上已经存在具有相同标签的实例，则不进行部署，避免了一个节点上部署多个相同的实例。**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

**注解：再创建3个Web服务的实例，同上面Redis的配置，首先确保两个Web不会部署到相同的节点，然后在应用Pod间亲和策略，优先在有Redis服务的节点上部署Web。**