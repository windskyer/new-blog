---
title: pod服务质量qos解析
date: 2019-01-21 08:50:58
tags: [pod]
categories: Kubernetes
---

# Pod 配置服务质量等级

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

Pod 可以配置特定的服务质量（QoS）等级。Kubernetes 使用 QoS 等级来确定何时调度和终结 Pod 。

## QoS 等级

当 Kubernetes 创建一个 Pod 时，它就会给这个 Pod 分配一个 QoS 等级：

>- Guaranteed
>- Burstable
>- BestEffort

<!-- more -->

## 创建一个 Pod 并分配 QoS 等级为 Guaranteed

想要给 Pod 分配 QoS 等级为 Guaranteed:

>- Pod 里的每个容器都必须有内存限制和请求，而且必须是一样的。
>- Pod 里的每个容器都必须有 CPU 限制和请求，而且必须是一样的。
>- pod 中的mem， cpu 的 limit 和request 值必须一样。

这是一个含有一个容器的 Pod 的配置文件。这个容器配置了内存限制和请求，都是200MB。它还有 CPU 限制和请求，都是700 millicpu:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

查看 Pod 的详细信息:

```bash
kubectl get pod qos-demo --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给 Pod 配置的 QoS 等级为 Guaranteed 。也验证了容器的内存和 CPU 的限制都满足了它的请求。

```yaml
spec:
  containers:
    ...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
...
  qosClass: Guaranteed
```

## 创建一个 Pod 并分配 QoS 等级为 BestEffort

要给一个 Pod 配置 BestEffort 的 QoS 等级, Pod 里的容器必须没有任何内存或者 CPU　的限制或请求。

>- 不配置容器 resource 信息

下面是一个　Pod　的配置文件，包含一个容器。这个容器没有内存或者 CPU 的限制或者请求：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```

查看 Pod 的详细信息:

```bash
kubectl get pod qos-demo-3 --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给 Pod 配置的 QoS 等级是 BestEffort.

```yaml
spec:
  containers:
    ...
    resources: {}
  ...
  qosClass: BestEffort
```

## 创建一个 Pod 并分配 QoS 等级为 Burstable

当出现下面的情况时，则是一个 Pod 被分配了 QoS 等级为 Burstable :

>- 该 Pod 不满足 QoS 等级 Guaranteed 的要求。
>- Pod 里至少有一个容器有内存或者 CPU 请求。
>- 不满足Guaranteed，BestEffort 的pod 都是Burstable。

这是 Pod 的配置文件，里面有一个容器。这个容器配置了200MB的内存限制和100MB的内存申请。

```yaml
qos-pod-2.yaml  Copy qos-pod-2.yaml to clipboard
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

查看 Pod 的详细信息:

```bash
kubectl get pod qos-demo-2 --namespace=qos-example --output=yaml
```

输出显示了 Kubernetes 给这个 Pod 配置了 QoS 等级为 Burstable.

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: qos-demo-2-ctr
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
...
  qosClass: Burstable
```

## 总结

在k8s中，会根据pod的limit 和 requests的配置将pod划分为不同的qos类别：

>- Guaranteed
>- Burstable
>- BestEffort

**当机器可用资源不够时，kubelet会根据qos级别划分迁移驱逐pod。被驱逐的优先级：BestEffort > Burstable > Guaranteed**