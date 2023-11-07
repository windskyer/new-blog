---
title: Istio 部署与安装
categories: Kubernetes
sage: false
date: 2020-09-21 10:27:42
tags: istio
---

## Istio部署

目前 Istio 支持三种方式进行部署：

>1. 使用Helm自定义安装 （已弃用）
>2. 使用Operator安装 （实验阶段）
>3. 使用istioctl工具安装 （官方推荐）

<!-- more -->

### Istio单集群模式部署

Istio最简部署是指单一网络，单一集群，单一控制平面的部署模式，Istio最简部署方案不具备高可用性，故障隔离，故障转移等特性

![7](Istio-架构与概念/7.png)

#### 使用 istioctl 安装

```bash
# 下载 istio 安装包
$ curl -L https://istio.io/downloadIstio | sh -
 
# 将 istio 安装包中 istioctl 二进制文件添加到系统路径中
$ cd istio-1.6.5 && export PATH=$PWD/bin:$PATH
 
# 使用istio内置的配置模板快速安装
$ istioctl install --set profile=default --set hub=hub.docker.com/docker/istio  # 多集群部署时需要安装istiocoredns组件 --set addonComponents.istiocoredns.enabled=true
 
# 基于默认安装istio后，自定义修改配置项
$ istioctl manifest apply --set values.<item>=<value>
 
 
# 设置istioctl自动补全
$ yum install bash-completion -y
$ source istio-1.6.5/tools/istioctl.bash
$ source /usr/share/bash-completion/bash_completion
```

#### 使用 yaml manifest 安装

```bash
# 手动 创建 istio-system 命名空间
$ kubectl create namespace istio-system
 
# 使用istioctl导出的配置yaml自定义安装
$ istioctl manifest generate --set values.global.jwtPolicy=first-party-jwt --set profile=default > istio-default.yaml
# 验证安装
$ istioctl verify-install -f istio-default.yaml
```

***注意： 使用 kubectl apply yaml文件的方式安装Istio 官方暂未测试，同时可能由于执行顺序问题导致安装短暂失败，并且istio配置变更时不会自动清理相关资源，不推荐使用***

### Istio多集群部署

由于Istio最简模式缺乏服务高可用，故障转移，故障隔离等特性，因此如果对服务有高可用要求时则需要使用多集群的方式部署；多集群部署比单集群可以提供更多的能力：

>1. 故障隔离和故障转移：当 cluster-1 下线，业务将转移至 cluster-2
>2. 位置感知路由和故障转移：将请求发送到最近的服务
>3. 多种控制平面模型：支持不同级别的可用性
>4. 团队或项目隔离：每个团队仅运行自己的集集群

![8](Istio-架构与概念/8.png)

#### 多集群多控制平面

![9](Istio-架构与概念/9.png)

##### 前提条件

>1. 两个以上 Kubernetes 集群，且版本为：1.15, 1.16, 1.17, 1.18
>2. 有权限在 每个 Kubernetes 集群上，部署 Istio 控制平面
>3. 每个集群 istio-ingressgateway 服务的 IP 地址，必须允许其它集群访问
>4. 多集群使用同一个根CA证书。跨集群的服务通信必须使用双向 TLS 连接

##### 部署步骤

创建CA 用于多集群间TLS通信，在全部Kubernetes 集群中执行以下操作

```bash
$ kubectl create namespace istio-system
$ kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-ch
```

安装Istio

```bash
$ istioctl install \
    -f manifests/examples/multicluster/values-istio-multicluster-gateways.yaml
```

配置DNS

```yaml
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    global:53 {
        errors
        cache 30
        proxy . $(kubectl get svc -n istio-system istiocoredns -o jsonpath={.spec.clusterIP})
    }
EOF
```

***备注：cluster-1 如果需要访问cluster-2的服务时，需要创建 ServiceEntry，例如：在cluster-1 中需要访问 cluster-2的bar命名空间下的httpbin服务，具体ServiceEntry声明如下***

```yaml
$ kubectl apply --context=$CTX_CLUSTER1 -n foo -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-bar
spec:
  hosts:
  # must be of form name.namespace.global
  - httpbin.bar.global
  # Treat remote cluster services as part of the service mesh
  # as all clusters in the service mesh share the same root of trust.
  location: MESH_INTERNAL
  ports:
  - name: http1
    number: 8000
    protocol: http
  resolution: DNS
  addresses:
  # the IP address to which httpbin.bar.global will resolve to
  # must be unique for each remote service, within a given cluster.
  # This address need not be routable. Traffic for this IP will be captured
  # by the sidecar and routed appropriately.
  - 240.0.0.2
  endpoints:
  # This is the routable address of the ingress gateway in cluster2 that
  # sits in front of sleep.foo service. Traffic from the sidecar will be
  # routed to this address.
  - address: ${CLUSTER2_GW_ADDR}
    ports:
      http1: 15443 # Do not change this port value
EOF
```
多集群多控制平面本质多个独立的集群，使用global域名进行服务发现，通过注册ServiceEntry访问“外部”集群服务的方式调用远程服务并使用统一的CA建立双向TLS通信保证传输安全

#### 多集群共享控制平面

![10](Istio-架构与概念/10.png)

#### 前提条件

>1. 两个以上 Kubernetes 集群，且版本高于1.15
>2. 全部 Kubernetes API server 必须可以互相访问
>3. （如果多个集群处于同一网络时）必须满足集群Pod CIDR 和Service CIDR 在多集群网络中必须唯一不能有重叠 并且所有Pod的网络必须可以互通
>4. （如果多个集群处于不同网络时）要求 istio-ingressgateway 必须可以被其他集群访问

##### 部署步骤
创建CA 用于多集群间TLS通信，在全部Kubernetes 集群中执行以下操作

```bash
$ kubectl create namespace istio-system
$ kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem

```

对集群及网络命名

```bash

$ export MAIN_CLUSTER_NAME=main0
$ export REMOTE_CLUSTER_NAME=remote0
 
 
$ export MAIN_CLUSTER_NETWORK=network1
$ export REMOTE_CLUSTER_NETWORK=network2
 
 
# 多集群单网络
$ export MAIN_CLUSTER_NETWORK=network1
$ export REMOTE_CLUSTER_NETWORK=network1
```

通过使用负载均衡的方式，将主集群 Istiod 服务发现暴露给其他集群

```bash
$ cat <<EOF> istio-main-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      multiCluster:
        clusterName: ${MAIN_CLUSTER_NAME}
      network: ${MAIN_CLUSTER_NETWORK}
 
      # Mesh network configuration. This is optional and may be omitted if
      # all clusters are on the same network.
      meshNetworks:
        ${MAIN_CLUSTER_NETWORK}:
          endpoints:
          - fromRegistry:  ${MAIN_CLUSTER_NAME}
          gateways:
          - registry_service_name: istio-ingressgateway.istio-system.svc.cluster.local
            port: 443
 
        ${REMOTE_CLUSTER_NETWORK}:
          endpoints:
          - fromRegistry: ${REMOTE_CLUSTER_NAME}
          gateways:
          - registry_service_name: istio-ingressgateway.istio-system.svc.cluster.local
            port: 443
 
  # Change the Istio service `type=LoadBalancer` and add the cloud provider specific annotations. See
  # https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer for more
  # information. The example below shows the configuration for GCP/GKE.
  components:
    pilot:
      k8s:
        service:
          type: LoadBalancer
        service_annotations:
          cloud.google.com/load-balancer-type: Internal
EOF
 
$ istioctl install -f istio-main-cluster.yaml
```

```bash
$ export ISTIOD_REMOTE_EP=$(kubectl get svc -n istio-system --context=${MAIN_CLUSTER_CTX} istiod -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ echo "ISTIOD_REMOTE_EP is ${ISTIOD_REMOTE_EP}"
```

配置安装远程集群

```bash
cat <<EOF> istio-remote0-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      # The remote cluster's name and network name must match the values specified in the
      # mesh network configuration of the main cluster.
      multiCluster:
        clusterName: ${REMOTE_CLUSTER_NAME}
      network: ${REMOTE_CLUSTER_NETWORK}
 
      # Replace ISTIOD_REMOTE_EP with the the value of ISTIOD_REMOTE_EP set earlier.
      remotePilotAddress: ${ISTIOD_REMOTE_EP} 
 
 
  ## The istio-ingressgateway is not required in the remote cluster if both clusters are on
  ## the same network. To disable the istio-ingressgateway component, uncomment the lines below.
  #
  # components:
  #  ingressGateways:
  #  - name: istio-ingressgateway
  #    enabled: false
EOF
 
$ istioctl install -f istio-remote0-cluster.yaml
```