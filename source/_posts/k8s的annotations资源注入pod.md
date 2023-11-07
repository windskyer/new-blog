---
title: k8s的annotations资源注入pod
categories: Kubernetes
sage: false
date: 2023-03-27 16:01:01
tags: k8s
---

## 背景目的

当容器使用ovn网络的时候，需要给容器里面注入ovn 分配对应的vf网卡的ip信息
<!-- more -->

## 注入方法

给每个容器挂在volume，注入vf信息固定写法

    #yaml容器段，定义挂载目录地址
        volumeMounts:
        - name: vfus
          mountPath: /opt/yusur_ovn
    #yaml pod段，定义volume类容
    volumes:
        - name: vfus
          downwardAPI:
            items:
              - path: "config.ini"
                fieldRef:
                  fieldPath: metadata.annotations['k8s.ovn.org/vfus-networks']

## Pods资源

yaml 案例:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx1-ovn-pod
      namespace: default
      annotations:
        v1.multus-cni.io/default-network: calico-net
        k8s.v1.cni.cncf.io/networks: kube-system/ovn-net@eth1 #配置辅助cni为ovn
      labels:
        app: nginx1-ovn-pod
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx1-ovn-pod
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
        volumeMounts:
        - name: vfus
          mountPath: /opt/yusur_ovn
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      restartPolicy: Always
      volumes:
        - name: vfus
          downwardAPI:
            items:
              - path: "config.ini"
                fieldRef:
                  fieldPath: metadata.annotations['k8s.ovn.org/vfus-networks']

查看结果：

    kubectl exec -it nginx1-ovn-pod -- cat /opt/yusur_ovn/config.ini | jq
    {
      "port0": {
        "addr": "10.124.1.18",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }

## Deployments资源

yaml 案例:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: default
      name: nginx1-ovn-deploy
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx1-ovn-deploy
      template:
        metadata:
          labels:
            app: nginx1-ovn-deploy
          annotations:
            v1.multus-cni.io/default-network: calico-net 
            k8s.v1.cni.cncf.io/networks: kube-system/ovn-net@eth1 #配置辅助cni为ovn
        spec:
          nodeSelector:
            k8s.ovn.org/dpu-host: ""
          containers:
          - name: nginx1-ovn-deploy
            image: ubuntu:22.04-yusur
            imagePullPolicy: IfNotPresent
            resources:
              requests:
                yusur.tech/sriov_dpu: '1'
              limits:
                yusur.tech/sriov_dpu: '1'
            ports:
            - containerPort: 80
            volumeMounts:
            - name: vfus
              mountPath: /opt/yusur_ovn
          volumes:
           - name: vfus
             downwardAPI:
               items:
                 - path: "config.ini"
                   fieldRef:
                     fieldPath: metadata.annotations['k8s.ovn.org/vfus-networks']

查看结果：

    kubectl exec -it nginx1-ovn-deploy-c474975db-l642g -- cat /opt/yusur_ovn/config.ini | jq
    {
      "port0": {
        "addr": "10.124.1.17",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }

## Daemonsets资源

yaml 案例:

    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: nginx1-ovn-ds
      labels:
        app: nginx1-ovn-ds
    spec:
      selector:
        matchLabels:
          octopusexport: OctopusExport
      updateStrategy:
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: nginx1-ovn-ds
            octopusexport: OctopusExport
          annotations:
            v1.multus-cni.io/default-network: calico-net
            k8s.v1.cni.cncf.io/networks: kube-system/ovn-net@eth1
        spec:
          volumes:
            - name: vfus
              downwardAPI:
                items:
                  - path: config.ini
                    fieldRef:
                      fieldPath: 'metadata.annotations[''k8s.ovn.org/vfus-networks'']'
          containers:
            - name: nginx1-ovn-ds
              image: ubuntu:22.04-yusur
              imagePullPolicy: IfNotPresent
              ports:
                - name: server
                  containerPort: 80
                  protocol: TCP
              volumeMounts:
                - name: vfus
                  mountPath: /opt/yusur_ovn
                  subPath: ''
              resources:
                requests:
                  yusur.tech/sriov_dpu: '1'
                limits:
                  yusur.tech/sriov_dpu: '1'
              securityContext:
                privileged: true
                runAsNonRoot: false

查看结果：

    kubectl exec -it nginx1-ovn-ds-mt747 -- cat /opt/yusur_ovn/config.ini | jq
    {
      "port0": {
        "addr": "10.124.1.21",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }

## Statefulsets资源

yaml 案例:

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: nginx1-ovn-sts
      labels:
        app: nginx1-ovn-sts
    spec:
      selector:
        matchLabels:
          octopusexport: OctopusExport
      replicas: 1
      serviceName: nginx1-ovn-sts
      updateStrategy:
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: nginx1-ovn-sts
            octopusexport: OctopusExport
          annotations:
            v1.multus-cni.io/default-network: calico-net
            k8s.v1.cni.cncf.io/networks: kube-system/ovn-net@eth1
        spec:
          volumes:
            - name: vfus
              downwardAPI:
                items:
                  - path: config.ini
                    fieldRef:
                      fieldPath: 'metadata.annotations[''k8s.ovn.org/vfus-networks'']'
          containers:
            - name: nginx1-ovn-sts
              image: ubuntu:22.04-yusur
              imagePullPolicy: IfNotPresent
              ports:
                - name: server
                  containerPort: 80
                  protocol: TCP
              volumeMounts:
                - name: vfus
                  mountPath: /opt/yusur_ovn
              resources:
                requests:
                  yusur.tech/sriov_dpu: '1'
                limits:
                  yusur.tech/sriov_dpu: '1'
              securityContext:
                privileged: true
                runAsNonRoot: false

查看结果：

    kubectl exec -it nginx1-ovn-sts-0 -- cat /opt/yusur_ovn/config.ini | jq
    {
      "port0": {
        "addr": "10.124.1.22",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }

## 多容器注入

每个container 都需挂载volume

yaml 案例:

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx1-ovn-pod2
      namespace: default
      annotations:
        v1.multus-cni.io/default-network: calico-net
        k8s.v1.cni.cncf.io/networks: kube-system/ovn-net@eth1 #配置辅助cni为ovn
      labels:
        app: nginx1-ovn-pod2
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx1-ovn-pod2
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
        volumeMounts:
        - name: vfus
          mountPath: /opt/yusur_ovn
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'sleep 355555']
        imagePullPolicy: IfNotPresent
        name: nginx1-ovn-pod2-2
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 81
        volumeMounts:
        - name: vfus
          mountPath: /opt/yusur_ovn
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      restartPolicy: Always
      volumes:
        - name: vfus
          downwardAPI:
            items:
              - path: "config.ini"
                fieldRef:
                  fieldPath: metadata.annotations['k8s.ovn.org/vfus-networks']

查看结果：

    root@yusur-62:/home/leid/cni/test/vfus# kubectl exec -it nginx1-ovn-pod2 -c nginx1-ovn-pod2 -- cat /opt/yusur_ovn/config.ini | jq 
    {
      "port0": {
        "addr": "10.124.1.4",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }
    root@yusur-62:/home/leid/cni/test/vfus# kubectl exec -it nginx1-ovn-pod2 -c nginx1-ovn-pod2-2 -- cat /opt/yusur_ovn/config.ini | jq 
    {
      "port0": {
        "addr": "10.124.1.4",
        "netmask": "255.255.255.0",
        "broadcast": "10.124.1.255",
        "gateway": "10.124.1.1"
      }
    }

## 参考工具

yaml生成工具[https://k8syaml.com/](https://k8syaml.com/)
