---
title: Custom HPA水平自动伸缩
categories: Kubernetes
sage: false
date: 2020-10-31 14:30:56
tags: k8s
---

## 简介

Custom HPA 是Kubernetes中Pod水平自动伸缩的一种，不同于标准的HPA基于Pod的CPU使用率和内存使用量来控制工作负载中的副本数量，Custom HPA支持以自定义监控指标来实现对工作负载中副本数的控制

<!-- more -->

![1](Custom-HPA水平自动伸缩/1.png)

目前社区提供了基于Prometheus的Costom HPA方案，方案中包含以下组件：

>- Prometheus：云原生监控告警系统，用于周期性采集Pod中的自定义监控数据
>- Prometheus Adapter:  Prometheus适配器，用于将接收到custom metric api 请求转换成 promethus请求，并将prometheus 返回的数据转换成 custom metric api 定义的标准数据格式
>- Metrics aggreagtor：Kubernetes API 汇聚层中的一部分，用于将外部监控服务集成到 kube-apiserver中
>- Horizontal Pod AutoScaler: Pod水平自动伸缩功能的核心服务，周期性获取HPA实例中定义的监控指标数据，对比定义中的期望值，动态调整工作负载中的副本数量

## 基于Promethus的Custom HPA 部署实践

### 安装Helm V2

在Kubernetes集群内任意节点上执行如下命令，安装helm client

```bash
# 下载Helm二进制文件
$ wget https://get.helm.sh/helm-v2.16.1-linux-amd64.tar.gz
 
# 解压缩helm-v2.16.1-linux-amd64.tar.gz
$ tar zxvf helm-v2.16.1-linux-amd64.tar.gz
 
# 将Helm 二进制文件拷贝至 /usr/local/bin目录下
$ mv ~/linux-amd64/helm /usr/local/bin
```

安装Helm 服务端组件Tiller 

```bash
# 创建 ServiceAccount
$ kubectl create serviceaccount tiller --namespace=kube-system
 
# 赋予tiller admin权限
$ kubectl create clusterrolebinding tiller-admin --serviceaccount=kube-system:tiller --clusterrole=cluster-admin
 
# 使用 tiller ServiceAccount 部署安装 Tiller
$ helm init --service-account=tiller --tiller-image=hub.ksyun.com/docker/tiller:v2.16.1 --skip-refresh
 
# 查看tiller-deploy是否创建成功
$ kubectl get deploy tiller-deploy -n kube-system
```

helm 官方源 https://kubernetes-charts.storage.googleapis.com ，国内的某些机器无法访问，需要配置镜像源

```bash
# Azure 镜像
$ helm repo add stable http://mirror.azure.cn/kubernetes/charts/
$ helm repo add incubator http://mirror.azure.cn/kubernetes/charts-incubator/
 
# 更新源
$ helm repo update
```

### 开启Kubernetes API聚合层
在集群master节点修改 kube-apiserver.yaml 配置如下启动项

注意：K8S集群默认现已开启 Kubernetes API 聚合层，用户无需手动配置

```yaml
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
  containers:
  - command:
    ...
    # 加入以下配置
    - --requestheader-client-ca-file=/etc/kubernetes/ssl/aggregator/aggregator-ca.pem
    - --proxy-client-cert-file=/etc/kubernetes/ssl/aggregator/aggregator.pem
    - --proxy-client-key-file=/etc/kubernetes/ssl/aggregator/aggregator-key.pem
    - --requestheader-allowed-names=aggregator
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
```

## 部署Prometheus

```bash
# 创建命名空间
$ kubectl create namespace monitoring
 
# 创建StorageClass
$ kubectl apply -f storageclass.yaml
 
# 使用Kubernetes charts仓库中prometheus的chart包进行安装
$ helm install stable/prometheus --namespace monitoring --values prom-values.yaml --name prometheus
```

## 部署Prometheus Adapter

```bash
$ helm install stable/prometheus-adapter --values adapter-values.yaml --name prometheus-adapter --namespace monitoring
```

备注：启动时，--v=6 可以打印出adapter请求prometheus的query信息

验证Prometheus Adapter服务是否可用

```bash
$ kubectl get apiservice | grep prometheus-adapter
NAME                                   SERVICE                         AVAILABLE   AGE
v1beta1.custom.metrics.k8s.io          monitoring/prometheus-adapter   True        3h42m
```

## 自定义HPA功能验证

### 部署测试服务

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-prom-demo
spec:
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      prometheus.io/scrape: "true"
      prometheus.io/port: "80"
      prometheus.io/path: "/status/format/prometheus"
      labels:
        app: nginx-server
    spec:
      containers:
        - name: nginx-demo
          image: cnych/nginx-vts:v1.0
          resources:
            limits:
              cpu: 50m
            requests:
              cpu: 50m
          ports:
            - containerPort: 80
              name: http
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-prom-demo
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "80"
    prometheus.io/path: "/status/format/prometheus"
spec:
  ports:
    - port: 80
      targetPort: 80
      name: http
  selector:
    app: nginx-server
  type: NodePort
```

查询测试服务提供的自定义监控数据

```yaml
export DEMO_SVC_IP=$(kubectl get svc  hpa-prom-demo -o jsonpath='{.spec.clusterIP}')
export DEMO_SVC_PORT=$(kubectl get svc  hpa-prom-demo -o jsonpath='{.spec.ports[0].port}')
 
curl http://${DEMO_SVC_IP}:${DEMO_SVC_PORT}/status/format/prometheus | grep nginx_vts_server_requests_total
nginx_vts_server_requests_total{host="*",code="1xx"} 0
nginx_vts_server_requests_total{host="*",code="2xx"} 13328
nginx_vts_server_requests_total{host="*",code="3xx"} 0
nginx_vts_server_requests_total{host="*",code="4xx"} 0
nginx_vts_server_requests_total{host="*",code="5xx"} 0
nginx_vts_server_requests_total{host="*",code="total"} 13328
...
```

### 创建HPA

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-prom-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metricName: nginx_vts_server_requests_per_second     # 自定义指标项 —— nginx_vts_server每秒请求数
        targetAverageValue: 4000m                            # 期望平均每秒请求数量为 4000m 即 4个请求/秒
```

### 设置Prometheus Adapter查询规则

修改 prometheus adapter configmap 定义自定义指标查询规则，根据自定义监控指标需要配置指定查询规则，建议去掉冗余的查询规则

```yaml
$ kubectl edit cm prometheus-adapter -n monitoring
config.yaml: |
    rules:
    - seriesQuery: 'nginx_vts_server_requests_total'
      resources:
        overrides:
          kubernetes_namespace:
            resource: namespace
          kubernetes_pod_name:
            resource: pod
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>,code!="", kubernetes_pod_name!="",code!="total",host!="_"}[2m])) by (<<.GroupBy>>))
```

[配置参数使用说明](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config.md)
[配置范例](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md)

查询自定义监控指标项，验证配置的查询规则是否生效

```yaml
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/" | jq .
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/nginx_vts_server_requests_per_second",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/nginx_vts_server_requests_per_second",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
```

查询自定义监控指标数据

```yaml
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/nginx_vts_server_requests_per_second" | jq .
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/nginx_vts_server_requests_per_second"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "hpa-prom-demo-547b49c5c9-6dmgj",
        "apiVersion": "/v1"
      },
      "metricName": "nginx_vts_server_requests_per_second",
      "timestamp": "2020-08-13T06:50:22Z",
      "value": "33m",
      "selector": null
    },
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "hpa-prom-demo-547b49c5c9-8zdt4",
        "apiVersion": "/v1"
      },
      "metricName": "nginx_vts_server_requests_per_second",
      "timestamp": "2020-08-13T06:50:22Z",
      "value": "33m",
      "selector": null
    }
  ]
}
```

### 对测试服务进行模拟压力测试

```yaml
export SERVICE_CLUSTER_IP=$(kubectl get svc hpa-prom-demo -o jsonpath='{.spec.clusterIP}')
$ hey -z 10m -q 50 -c 10 -m GET  http://${SERVICE_CLUSTER_IP}
```
参数说明：

>- -z : 压测持续时间
>- -q : 单worker每秒请求数量(QPS)
>- -c : worker数量

查看测试服务deploy的副本数

```yaml
$ kubectl get deploy  hpa-prom-demo -w
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
hpa-prom-demo   2/4     2            2           2d21h
hpa-prom-demo   2/4     4            2           2d21h
hpa-prom-demo   3/4     4            3           2d21h
hpa-prom-demo   4/4     4            4           2d21h
hpa-prom-demo   4/8     4            4           2d21h
hpa-prom-demo   4/8     4            4           2d21h
hpa-prom-demo   4/8     8            4           2d21h
hpa-prom-demo   5/8     8            5           2d21h
hpa-prom-demo   6/8     8            6           2d21h
hpa-prom-demo   7/8     8            7           2d21h
hpa-prom-demo   8/8     8            8           2d21h
hpa-prom-demo   8/10    8            8           2d21h
hpa-prom-demo   8/10    8            8           2d21h
hpa-prom-demo   9/10    10           9           2d21h
hpa-prom-demo   10/10   10           10          2d21h
```

查看HPA事件信息

```yaml
$ kubectl get hpa nginx-custom-hpa
NAME               REFERENCE                  TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
nginx-custom-hpa   Deployment/hpa-prom-demo   96953m/4   2         10        10         47h
 
 
$ kubectl describe hpa nginx-custom-hpa
...
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from pods metric nginx_vts_server_requests_per_second
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age                  From                       Message
  ----    ------             ----                 ----                       -------
  Normal  SuccessfulRescale  47h                  horizontal-pod-autoscaler  New size: 2; reason: Current number of replicas below Spec.MinReplicas
  Normal  SuccessfulRescale  47h                  horizontal-pod-autoscaler  New size: 9; reason: All metrics below target
  Normal  SuccessfulRescale  47h                  horizontal-pod-autoscaler  New size: 7; reason: All metrics below target
  Normal  SuccessfulRescale  47h                  horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
  Normal  SuccessfulRescale  6m20s (x2 over 47h)  horizontal-pod-autoscaler  New size: 4; reason: pods metric nginx_vts_server_requests_per_second above target
  Normal  SuccessfulRescale  6m5s (x2 over 47h)   horizontal-pod-autoscaler  New size: 8; reason: pods metric nginx_vts_server_requests_per_second above target
  Normal  SuccessfulRescale  5m50s (x2 over 47h)  horizontal-pod-autoscaler  New size: 10; reason: pods metric nginx_vts_server_requests_per_second above targetnginx_vts_server_requests_per_second above target
  Normal   SuccessfulRescale             48m                   horizontal-pod-autoscaler  New size: 2; reason: All metrics below target

```