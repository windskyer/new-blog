---
title: golang依赖管理
categories: Kubernetes
sage: false
date: 2020-02-26 14:51:08
tags: go
---

# 背景

## govendor 缺点

govendor依赖管理太松散，同一个依赖项目，不同的组件引用，版本可以不一样，例如vendor.json：

```json
    {
         "checksumSHA1": "ehAUZgg3BT4gz3WA5B9l2o4NOHw=",
         "path": "k8s.io/apimachinery/pkg/api/equality",
         "revision": "035e418f1ad9b6da47c4e01906a0cfe32f4ee2e7",
         "revisionTime": "2019-07-31T12:28:47Z"
    },
    {
          "checksumSHA1": "bmva3UAPnGM9sI9Ap5hXRhlH4wA=",
          "path": "k8s.io/apimachinery/pkg/api/errors",
          "revision": "1f8faeb8119141131b81637c896fc4c30e7075ae",
          "revisionTime": "2019-07-30T15:53:30Z"
    },
```

<!-- more -->

这样，当有其他依赖如 k8s.io/client-go，也依赖 k8s.io/apimachinery 的不通版本时，会造成依赖冲突

## go mod 优点

>- 官方推出的依赖管理工具
>- 可以摆脱GOPATH
>- 可以通过代理下载墙外依赖

## 环境准备

>1. golang 升级到 1.13
>2. 设置环境变量：
```bash
    GOPROXY=https://goproxy.cn
    GOPRIVATE=*.flftuu.com
    GONOPROXY=*.flftuu.com
    GONOSUMDB=*.flftuu.com
    GO111MODULE=auto
```

# 初始化项目

>1. newgit上新建项目，例如：newgit.op.flftuu.com/my/my_project
>2. 本地新建项目根目录，并初始化 go mod    
```bash
    $ mkdir my_project 
    $ cd my_project
    $ go mod init newgit.op.flftuu.com/my/my_project
```
>3. 写完代码后，更新依赖并保存到vendor目录
```bash
    $ go mod tidy
    $ go mod vendor
```

>4. git 提交代码

## 从 govendor 改造
>1. 下载项目，并删除vendor目录
>2. 初始化 go mod
```bash
    $ cd my_project
    $ go mod init newgit.op.flftuu.com/my/my_project
    $ go mod tidy
```
>3. 添加依赖
```bash
$ go get k8s.io/api@kubernetes-1.14.0
$ go get -v -insecure newgit.op.flftuu.com/kce/appclient@develop  #添加非 https 服务的依赖，需要将 git 账号密码保存到 windows ，否则拉取会失败
```

>4. 保存依赖到vendor
```bash
    $ go mod vendor
```

## 注意事项

>1. 使用go mod管理的依赖，执行 go get 时需要指定版本，例如：
```bash
    以分支为版本：go get k8s.io/api@master
    以 tag 为版本：go get k8s.io/api@kubernetes-1.14.0
    以 commit 为版本: go get k8s.io/api@40a48860b5abbba9aa891b02b32da429b08d96a0
```
>2. 由于 client-go 对 kubernetes 项目兼容性不足，需要使用client-go对应的kubernetes相关依赖版本，详见 https://github.com/kubernetes/client-go#compatibility-matrix
>3. 本次改动，基于 kubernetes 的版本全部为 1.14 ，例如：
```bash
    k8s.io/apimachinery@kubernetes-1.14.0
    k8s.io/client-go@v11.0.0    
    k8s.io/api@kubernetes-1.14.0
    k8s.io/apiextensions-apiserver@kubernetes-1.14.0
    k8s.io/apiserver@kubernetes-1.14.0
    k8s.io/component-base@kubernetes-1.14.0
    k8s.io/apiextensions-apiserver@v0.0.0-20190315093550-53c4693659ed
    k8s.io/api
```