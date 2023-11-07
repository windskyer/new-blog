---
title: kubefed环境搭建
categories: Kubernetes
sage: false
date: 2020-03-01 15:13:12
tags:
---

# kubefed客户端安装
下载地址：https://github.com/kubernetes-sigs/kubefed/releases/tag/v0.2.0-alpha.1
```bash
curl -LO https://github.com/kubernetes-sigs/kubefed/releases/download/v0.2.0-alpha.1/kubefedctl-0.2.0-alpha.1-linux-amd64.tgz
tar -zxvf kubefedctl-*.tgz
chmod u+x kubefedctl
sudo mv kubefedctl /usr/local/bin/ # make sure the location is in the PATH
```

<!-- more -->

# 使用helm安装kubefed控制面服务
>1. 下载并安装helm命令行
>2. 初始化：

helm init --client-only
因 https://kubernetes-charts.storage.googleapis.com 网络不通，需要手动创建$HOME/.helm/repository目录下把helm的repositories.yaml文件新建即可，参考wiki:Helm 基本使用

helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts

helm repo list 能看到已添加的库

```bash
[root@vm192-168-0-203 cache]# helm repo list
NAME            URL
stable          https://kubernetes-charts.storage.googleapis.com
hub             https://hub.kce.ksyun.com/chartrepo/zhaoqi
kubefed-charts  https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
```


helm search kubefed 查询kubefed chart包

```bash
[root@vm192-168-0-203 cache]# helm search kubefed
WARNING: Repo "stable" is corrupt or missing. Try 'helm repo update'.
NAME                            CHART VERSION   APP VERSION DESCRIPTION
kubefed-charts/kubefed          0.2.0-alpha.1               KubeFed helm chart
kubefed-charts/federation-v2    0.0.10                      Kubernetes Federation V2 helm chart
```

helm install kubefed-charts/kubefed --name kubefed --version 0.2.0-alpha.1 --namespace kube-federation-system
```bash
[root@vm192-168-0-203 cache]# kubectl get crd | grep kubefed
clusterpropagatedversions.core.kubefed.io            2020-04-27T09:05:34Z
dnsendpoints.multiclusterdns.kubefed.io              2020-04-27T09:05:34Z
domains.multiclusterdns.kubefed.io                   2020-04-27T09:05:34Z
federatedclusterroles.types.kubefed.io               2020-04-27T09:05:34Z
federatedconfigmaps.types.kubefed.io                 2020-04-27T09:05:34Z
federateddeployments.types.kubefed.io                2020-04-27T09:05:34Z
federatedingresses.types.kubefed.io                  2020-04-27T09:05:34Z
federatedjobs.types.kubefed.io                       2020-04-27T09:05:34Z
federatednamespaces.types.kubefed.io                 2020-04-27T09:05:34Z
federatedreplicasets.types.kubefed.io                2020-04-27T09:05:34Z
federatedsecrets.types.kubefed.io                    2020-04-27T09:05:34Z
federatedserviceaccounts.types.kubefed.io            2020-04-27T09:05:34Z
federatedservices.types.kubefed.io                   2020-04-27T09:05:34Z
federatedservicestatuses.core.kubefed.io             2020-04-27T09:05:34Z
federatedtypeconfigs.core.kubefed.io                 2020-04-27T09:05:34Z
ingressdnsrecords.multiclusterdns.kubefed.io         2020-04-27T09:05:34Z
kubefedclusters.core.kubefed.io                      2020-04-27T09:05:34Z
kubefedconfigs.core.kubefed.io                       2020-04-27T09:05:34Z
propagatedversions.core.kubefed.io                   2020-04-27T09:05:34Z
replicaschedulingpreferences.scheduling.kubefed.io   2020-04-27T09:05:34Z
servicednsrecords.multiclusterdns.kubefed.io         2020-04-27T09:05:34Z

[root@vm192-168-0-203 cache]# kubectl get deploy -n kube-federation-system
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
kubefed-admission-webhook    1/1     1            1           69m
kubefed-controller-manager   2/2     2            2           69m
```

将两个集群加入联邦
```bash
[root@vm192-168-0-203 ~]# kubefedctl join --cluster-context context-1580a941-b081-4663-a998-4426ea561192 cluster1 --host-cluster-context context-1580a941-b081-4663-a998-4426ea561192
[root@vm192-168-0-203 ~]# kubefedctl join --cluster-context context-1580a941-b081-4663-a998-4426ea561192 cluster2 --host-cluster-context context-1580a941-b081-4663-a998-4426ea561192
[root@vm192-168-0-203 ~]# kubectl get kubefedcluster -n kube-federation-system
NAME       AGE   READY
cluster1   4h    True
cluster2   1s    True
```

联邦化的资源类型
```bash
[root@vm192-168-0-203 kubefed]# kubectl get FederatedTypeConfig -n kube-federation-system
NAME                                             AGE
clusterroles.rbac.authorization.k8s.io           1d
configmaps                                       1d
customresourcedefinitions.apiextensions.k8s.io   5h
deployments.apps                                 1d
ingresses.extensions                             1d
jobs.batch                                       3h
namespaces                                       1d
replicasets.apps                                 1d
secrets                                          1d
serviceaccounts                                  1d
services                                         1d
```


可以使用kubefedctl disable / enable 联邦化某种资源，包括crd资源类型
```bash
[root@vm192-168-0-203 kubefed]# kubefedctl disable job
Disabled propagation for FederatedTypeConfig "kube-federation-system/jobs.batch"
Verifying propagation controller is stopped for FederatedTypeConfig "kube-federation-system/jobs.batch"
Propagation controller for FederatedTypeConfig "kube-federation-system/jobs.batch" is stopped
federatedtypeconfig "kube-federation-system/jobs.batch" deleted
 
 
[root@vm192-168-0-203 kubefed]# kubefedctl enable job
customresourcedefinition.apiextensions.k8s.io/federatedjobs.types.kubefed.io updated
federatedtypeconfig.core.kubefed.io/jobs.batch created in namespace kube-federation-system
```