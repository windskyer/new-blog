---
title: Knative介绍
date: 2019-03-18 16:52:10
tags: serverless
categories: [Kubernetes]
---

# Knative 是什么

Knative 基于Kubernetes的平台，用来构建、部署和管理现代serverless工作负载。
该框架试图将开发云原生应用在三个领域的最佳实践结合起来，这三个领域指的是构建容器（和函数）、为工作负载提供服务（和动态扩展）以及事件。
Knative是由谷歌与Pivotal、IBM、Red Hat 和SAP紧密协作开发的。

<!-- more -->

# Knative组件

## Build

**描述:**

- Knative Build 扩展了 Kubernetes，并利用现有 Kubernetes 原语为你提供从源码运行集群容器构建的能力。
- 例如，你可以编写一个 Kubernetes-native 的 build 来从代码库获取源代码，将其构建到容器镜像中，然后运行该容器镜像。
- 虽然 Knative build 针对构建、测试和部署源代码进行了优化，但是您仍然需要负责开发相应的组件：

  - 从代码库中检索并获取源代码；
  - 在共享文件系统中运行多个顺序作业，比如：
    - 安装依赖；
    - 运行单元和集成测试；
  - 构建容器镜像；
  - 推送容器镜像到镜像仓库，或部署容器镜像到集群；

Knative Build 的目标是提高一个标准的、可移植的、可重用的、性能优化的方法，用户定义和运行集群容器镜像的构建。通过提供在 Kubernetes 上运行构建“枯燥而困难”的任务，Knative 使您不必开发和复制这些基于 Kubernetes 的开发过程。

尽管在现在，Knative build 并没有提供一套完整的 CI/CD 解决方案，但是它提供了一个较为低级的构建块，它是有意设计的，以便在更大的系统中实现基础和利用。

Build对象是Kubernetes集群的CRD（自定义资源）:

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: example-build-name
spec:
  serviceAccountName: build-auth-example
  source:
    git:
      url: https://github.com/example/build-example.git
      revision: master
  steps:
  - name: ubuntu-example
    image: ubuntu
    args: ["ubuntu-build-example", "SECRETS-example.md"]
  steps:
  - image: gcr.io/example-builders/build-example
    args: ['echo', 'hello-example', 'build']
  steps:
  - name: dockerfile-pushexample
    image: gcr.io/example-builders/push-example
    args: ["push", "${IMAGE}"]
    volumeMounts:
    - name: docker-socket-example
      mountPath: /var/run/docker.sock
  volumes:
  - name: example-volume
    emptyDir: {}
```

**Build对象:**

- 一个Build对象可以包含多个steps，每个step指定一个Builder。（step可以近似理解为 init container）
- 一个Build对象是一种创建出来完成任何任务的容器镜像，无论这是流程中的单个步骤，还是整个流程本身。
- Build对象中的steps可以推送到repository。
- 使用BuildTemplate可定义可重用模板。
- Build对象中的source可以被定义为挂载到Kubernete volumes，source支持git repository、Google Cloud Storage、任意容器镜像。
- 使用ServiceAccount+k8s Secrets进行认证。

## Serving

**描述:**

Knative Serving 建立在 Kubernetes 和 Istio 之上，以支持部署和服务无服务函数和应用。Knative Serving 项目提供了包含以下中间件原语的功能：

- 无服务容器的快速部署；
- 自动扩容，自动缩容乃至到0；
- 基于 Istio 组件的路由和网络编程；
- 已部署代码和配置的时间点快照；

**resources:**

- Service: service.serving.knative.dev
- Route: route.serving.knative.dev
- Configuration: configuration.serving.knative.dev
- Revision: revision.serving.knative.dev

**resource关系图:**
![knative-serving-object_model.png](knative/knative-serving-object_model.png)

## Eventing

**设计目标:**
Knative Eventing 是用来解决什么问题的？

- 在开发期间，Services 都是松耦合的，并且独立的部署在各种平台上 (Kubernetes, VMs, SaaS or FaaS)。
- 生产者能在消费者监听之前生成事件，消费者也能在生产者产生事件之前订阅感兴趣的事件。
- Services 能在以下两种场景下被连接 (be connected to) 以创建新的应用：
  - 不修改生产者或消费者。
  - 有能力从特定的生产者中选择特定的事件子集。

**从架构上分离出三个抽象组件:**

Buses

- Buses 提供了基于 Kafka 或 NATS 等消息总线的 k8s-native 的抽象。说白了就是 pubsub 机制。事件被发布到一个订阅通道，之后被路由到订阅的各方。
- Knative Eventing 目前支持三种 buses：Stub, Kafka, GCP PubSub。

Sources

- Sources 提供了一个类似的抽象层，用于从 Kubernetes外部提供数据源，并将它们路由到集群，以提要的形式表示。
- 目前提供三种源：K8sevents, GitHub, GCP PubSub。

Flows

- Flow是在事件中最顶层的用户面对的概念;它描述了从外部事件源到对事件作出反应的目的地的所需路径。

**抽象架构图:**
![eventing-concepts.png](knative/eventing-concepts.png)