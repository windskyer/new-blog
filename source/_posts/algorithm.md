---
title: k8s调度规则
date: 2019-02-28 15:33:40
tags: k8s
categories: Kubernetes
---

# 默认调度规则

```go
// 默认过滤规则14个
NoVolumeZoneConflictPred
MaxEBSVolumeCountPred
MaxGCEPDVolumeCountPred
MaxAzureDiskVolumeCountPred
MaxCSIVolumeCountPred
MatchInterPodAffinityPred
NoDiskConflictPred
GeneralPred
CheckNodeMemoryPressurePred
CheckNodeDiskPressurePred
CheckNodePIDPressurePred
CheckNodeConditionPred
PodToleratesNodeTaintsPred
CheckVolumeBindingPred

// 过滤规则
factory.RegisterFitPredicate("PodFitsPorts", predicates.PodFitsHostPorts)//这个只是为了兼容旧版本，所以仍旧保留，较新的版本已经用PodFitsHostPorts这个名字取代了PodFitsPorts，也就是下面的一个预选算法
factory.RegisterFitPredicate(predicates.PodFitsHostPortsPred, predicates.PodFitsHostPorts)
factory.RegisterFitPredicate(predicates.PodFitsResourcesPred, predicates.PodFitsResources)
factory.RegisterFitPredicate(predicates.HostNamePred, predicates.PodFitsHost)
factory.RegisterFitPredicate(predicates.MatchNodeSelectorPred, predicates.PodMatchNodeSelector)
```

<!-- more -->

## DefaultProvider 调度检查资源总结

> 注意：资源调度集合有两种【ClusterAutoscalerProvider | DefaultProvider】默认DefaultProvider
> The scheduling algorithm provider to use, one of: ClusterAutoscalerProvider | DefaultProvider (default "DefaultProvider")

### CheckNodeConditionPred （开启TaintNodesByCondition（node节点自动污点功能）功能将失效）

```yaml
# node节点环境情况查看
# kubectl  get node 10.0.99.14 -o yaml

  conditions:
  - lastHeartbeatTime: 2018-09-12T07:13:41Z
    lastTransitionTime: 2018-08-17T03:44:43Z
    message: kubelet has sufficient disk space available
    reason: KubeletHasSufficientDisk
    status: "False"
    type: OutOfDisk
  - lastHeartbeatTime: 2018-09-12T07:13:41Z
    lastTransitionTime: 2018-08-22T06:09:58Z
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
  - lastHeartbeatTime: 2018-09-12T07:13:41Z
    lastTransitionTime: 2018-08-22T06:09:58Z
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
  - lastHeartbeatTime: 2018-09-12T07:13:41Z
    lastTransitionTime: 2018-08-23T10:09:44Z
    message: kubelet is posting ready status
    reason: KubeletReady
    status: "True"
    type: Ready
```

> 节点状态ready
> 磁盘足够
> 网络是否正确配置
> 是否可以调度unschedulable: true( kubectl cordon node 设置不允许调度/uncordon允许调度)

### CheckNodeUnschedulablePred(需要开启TaintNodesByCondition功能)

```yaml
# kubectl  get node 10.0.99.14 -o yaml

spec:
  podCIDR: 172.16.1.0/24
  taints:
  - effect: NoSchedule
    key: node.kubernetes.io/unschedulable
    timeAdded: 2019-02-21T06:58:24Z
  unschedulable: true
```

> 检查node 节点taints node.kubernetes.io/unschedulable=NoSchedule 污点与pod 中Tolerations 是否设置容忍改污点

### GeneralPred

```yaml
# kubectl  get node 10.0.99.14 -o yaml

  allocatable:
    cpu: "1"
    ephemeral-storage: "18902281390"
    memory: 1781448Ki
    pods: "110"
```

```bash
# cpu,mem,storage 资源结算规则
const (
    // CPU, in cores. (500m = .5 cores)
    ResourceCPU ResourceName = "cpu"
    // Memory, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    ResourceMemory ResourceName = "memory"
    // Volume size, in bytes (e,g. 5Gi = 5GiB = 5 * 1024 * 1024 * 1024)
    ResourceStorage ResourceName = "storage"
    // Local ephemeral storage, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    // The resource name for ResourceEphemeralStorage is alpha and it can change across releases.
    ResourceEphemeralStorage ResourceName = "ephemeral-storage"
)
```

> 节点所允许分配的最大pod数量默认是110
> pod requests: cpu,mem,stroage 和node 存在的资源相加 小于 node 上能分配的资源
> 扩展资源的对比(具体yaml 的写法见wiki)

### HostNamePred

```go
// PodFitsHost checks if a pod spec node name matches the current node.
func PodFitsHost(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
    if len(pod.Spec.NodeName) == 0 {
        return true, nil, nil
    }
    node := nodeInfo.Node()
    if node == nil {
        return false, nil, fmt.Errorf("node not found")
    }
    if pod.Spec.NodeName == node.Name {
        return true, nil, nil
    }
    return false, []algorithm.PredicateFailureReason{ErrPodNotMatchHostName}, nil
}

```yaml
  dnsPolicy: ClusterFirst
  hostNetwork: true
  nodeName: 10.0.99.12
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  terminationGracePeriodSeconds: 30
```

> 指定node 节点调度 (在yaml文件中填写nodeName 字段)

## PodFitsHostPortsPred

> 获取node中所有的pod的hostip(hostport/protocol) 端口与所要创建的pod 中的 hostip(hostport/protocol) 进行对比
> eg： node 节点上有pod的端口设置成0.0.0.0(80/tcp) 如果现需创建pod 需要0.0.0.0(80/tcp)端口，则排除改node 节点

## MatchNodeSelectorPred

```yaml
#pod
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
          - key: kubernetes.io/e2e-az-num
            operator: In
            values:
            - az1
            - az2
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
```

> 检查pod 中设置的NodeSelector 选项
> 检查pod 中设置的 affinity.NodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution 亲和性
> nodeSelectorTerms 和 MatchFields 可以配置多个，但调度时候如果一个匹配则为调度成功（网上说法为全部匹配成功才算调度成功，源码显示为只需成功匹配一个就行）

## PodFitsResourcesPred

> 节点所允许分配的最大pod数量默认是110
> pod requests: cpu,mem,stroage 和node 存在的资源相加 小于 node 上能分配的资源
> 扩展资源的对比(如GPU  资源 nvidia.com/gpu， 有效资源key 中必须包含 "/"或者 不能以kubernetes.io/开头或者requests.开头)

## NoDiskConflictPred

> 检查在此主机上是否存在卷冲突。如果这个主机已经挂载了卷，其它同样使用这个卷的Pod不能调度到这个主机上。GCE,        Amazon EBS, and Ceph RBD使用的规则如下：
> GCE允许同时挂载多个卷，只要这些卷都是只读的。
> Amazon EBS不允许不同的Pod挂载同一个卷。
> Ceph RBD不允许任何两个pods分享相同的monitor，match pool和 image。
> ISCSI允许同时挂载多个卷，只要这些卷都是只读的。

## PodToleratesNodeTaintsPred (需要开启TaintNodesByCondition功能)

> 只检查pod 与 node 中的 NoSchedule， NoExecute 类型的容忍度和污点

```yaml
      tolerations:
      - operator: Exists
```

> 注解：如上是容忍所有污点
> 注解： NoSchedule 禁止调度， NoExecute 禁止调度，并驱逐调度上的pod， PreferNoSchedule 尽量不要调度上

## PodToleratesNodeNoExecuteTaintsPred

> 检查pod 是否容忍node 节点上的NoExecute 类型的污点
> 在DaemonSetsController 控制器中运用到，如果ds 中pod 没有设置容忍NoExecute 类型的污点，pod 将会调度不上去

## CheckNodeLabelPresencePred

```go
// NodeLabelChecker contains information to check node labels for a predicate.
type NodeLabelChecker struct {
    labels   []string
    presence bool
}
```

> 检查节点上是否存在所有指定的标签，无论它们的值如何
> 如果“presence”为false，则如果任何请求的标签与任何节点的标签匹配，则返回false，否则返回true。
> 如果“presence”为true，则如果任何请求的标签与任何节点的标签不匹配，则返回false，否则返回true。
> “presence”为false，node 节点中的label 与kube-scheduler自定的label 相匹配则 不让pod调度上
> “presence”为true，node 节点中的label 与kube-scheduler自定的label 相匹配则 让pod调度上

## CheckServiceAffinityPred

> 需要自行配置调度Policy策略中的PredicateArgument.ServiceAffinity 标签
> 为属于同一service 的pod 调度到同一组node 上

## MaxEBSVolumeCountPred

> 检查节点aws ebs 盘的个数限制 默认限制39个
> 如果设置FeatureGate（AttachVolumeLimit）功能则最大限制为node 节点所设置的值

## MaxGCEPDVolumeCountPred

> 检查节点gce 类型盘的个数限制 默认限制16个
> 如果设置FeatureGate（AttachVolumeLimit）功能则最大限制为node 节点所设置的值

## MaxAzureDiskVolumeCountPred

> 检查节点 Azure 类型盘的个数限制 默认限制16个
> 如果设置FeatureGate（AttachVolumeLimit）功能则最大限制为node 节点所设置的值

## CheckVolumeBindingPred (需要开启VolumeScheduling功能)>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
  annotations:
    "volume.alpha.kubernetes.io/node-affinity": '{
      "requiredDuringSchedulingIgnoredDuringExecution": {
        "nodeSelectorTerms": [
          { "matchExpressions": [
            { "key": "kubernetes.io/hostname",
              "operator": "In",
              "values": ["example-node"]
            }
          ]}
         ]}
        }'
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disk
```

>1. 检查pod 中所有pvc类型卷是否挂载(根据pvc 中的 pv.kubernetes.io/bind-completed 信息判断)
>2. 如果没有挂载并且开启VolumeScheduling功能和设置StorageClassName 类型并且绑定模式(volumeBindingMode) 为 WaitForFirstConsumer（如果volume.kubernetes.io/selected-node为true 则不进行预挂载）， 则pod 中的pvc 可以在pod 创建后进行绑定pv或者 StorageClass。
>3. 如果pod 中指定 pvc不满足2的要求并且没有绑定 pv 或者StorageClass 则调度失败。
>4. 对已经绑定的pvc检查其绑定的pv 指定的亲和性，如上yaml案例 如node 不满足pv的亲和性则pod 不会调度到该node上。
>5. 对没进行绑定的pvc 则根据其指定的StorageClass 匹配合适的pv 进行绑定

## NoVolumeZoneConflictPred

>1. 检查node 的 failure-domain.beta.kubernetes.io/zone 和 failure-domain.beta.kubernetes.io/region 标签
>2. 如果pod 的pv 中设置failure-domain.beta.kubernetes.io/zone, failure-domain.beta.kubernetes.io/region 标签，则只能调度到相同标签的node上
>3. 如果pod 的pv 中设置failure-domain.beta.kubernetes.io/zone, failure-domain.beta.kubernetes.io/region 标签，则调度该node 成功
>4. 改功能可用于 ebs 等区分region 和zone 的存储上

## CheckNodeMemoryPressurePred （开启TaintNodesByCondition（node节点自动污点功能）功能将失效）

```yaml
# kubectl get node 1.1.1.1 -o yaml
  conditions:
  - lastHeartbeatTime: 2019-02-28T10:17:13Z
    lastTransitionTime: 2018-12-13T08:46:38Z
    message: kubelet has sufficient memory available
    reason: KubeletHasSufficientMemory
    status: "False"
    type: MemoryPressure
```

>1. 检查pod的qos 是否为BestEffort
>2. qos 是 BestEffort 则检查node 节点内存是否压力过大，反之则跳过检查
>3. node 节点内存压力情况可根据node 详情查看

## CheckNodePIDPressurePred （开启TaintNodesByCondition（node节点自动污点功能）功能将失效）

```yaml
# kubectl get node 1.1.1.1 -o yaml
  conditions:
  - lastHeartbeatTime: 2019-02-28T10:17:13Z
    lastTransitionTime: 2018-12-13T08:46:38Z
    message: kubelet has sufficient PID available
    reason: KubeletHasSufficientPID
    status: "False"
    type: PIDPressure
```

>1. 则检查node 节点pid是否压力过大，反之则跳过检查
>2. node 节点pid压力情况可根据node 详情查看

## CheckNodeDiskPressurePred （开启TaintNodesByCondition（node节点自动污点功能）功能将失效）

```yaml
# kubectl get node 1.1.1.1 -o yaml
  - lastHeartbeatTime: 2019-02-28T10:17:13Z
    lastTransitionTime: 2018-12-13T08:46:38Z
    message: kubelet has no disk pressure
    reason: KubeletHasNoDiskPressure
    status: "False"
    type: DiskPressure
```

>1. 则检查node 节点disk是否压力过大，反之则跳过检查
>2. node 节点disk压力情况可根据node 详情查看

## MatchInterPodAffinityPred

> 检查pod 的亲和性设置, 与node 标签进行匹配

## NoVolumeZoneConflict

> 检查给定的zone限制前提下，检查如果在此主机上部署Pod是否存在卷冲突。假定一些volumes可能有zone调度约束， VolumeZonePredicate根据volumes自身需求来评估pod是否满足条件。必要条件就是任何volumes的zone-labels必须与节点上的zone-labels完全匹配。节点上可以有多个zone-labels的约束（比如一个假设的复制卷可能会允许进行区域范围内的访问）。目前，这个只对PersistentVolumeClaims支持，而且只在PersistentVolume的范围内查找标签。处理在Pod的属性中定义的volumes（即不使用PersistentVolume）有可能会变得更加困难，因为要在调度的过程中确定volume的zone，这很有可能会需要调用云提供商。

## MaxCSIVolumeCountPred

> 如果设置FeatureGate（AttachVolumeLimit） 功能则最大限制为node 节点所设置的值
> 开启AttachVolumeLimit 功能并且指定了 某种类型的csi dirver （eg: attachable-volumes-csi-xx）则xx 类型csi 卷在节点上不得超过指定的limits限制
