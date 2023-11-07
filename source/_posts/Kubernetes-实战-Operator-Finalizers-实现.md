---
title: Kubernetes 实战-Operator Finalizers 实现
categories: Kubernetes
sage: false
date: 2019-12-13 17:31:01
tags: k8s
---

## 背景

最近在写 k8s Operator，在看示例的时候看到 controller 都会设置 Finalizers，今天来聊一聊 Finalizers 和相关实现。

<!-- more -->

## Finalizers

Finalizers 允许 Operator 控制器实现异步的 pre-delete hook。比如你给 API 类型中的每个对象都创建了对应的外部资源，你希望在 k8s 删除对应资源时同时删除关联的外部资源，那么可以通过 Finalizers 来实现。

Finalizers 是由字符串组成的列表，当 Finalizers 字段存在时，相关资源不允许被强制删除。存在 Finalizers 字段的的资源对象接收的第一个删除请求设置 metadata.deletionTimestamp 字段的值， 但不删除具体资源，在该字段设置后， finalizer 列表中的对象只能被删除，不能做其他操作。

当 metadata.deletionTimestamp 字段非空时，controller watch 对象并执行对应 finalizers 的动作，当所有动作执行完后，需要清空 finalizers ，之后 k8s 会删除真正想要删除的资源。

## Operator finalizers 使用

介绍了 Finalizers 概念，那么我们来看看在 Operator 中如何使用，在 Operator Controller 中，最重要的逻辑就是 Reconcile 方法，finalizers 也是在 Reconcile 中实现的。要注意的是，设置了 Finalizers 会导致 k8s 的 delete 动作转为设置 metadata.deletionTimestamp 字段，如果你通过 kubectl get 命令看到资源存在这个字段，则表示资源正在删除（deleting）。

有以下几点需要理解：

    1. 如果资源对象未被删除且未设置 finalizers，则添加 finalizer并更新 k8s 资源对象；
    2. 如果正在删除资源对象并且 finalizers 仍然存在于 finalizers 列表中，则执行 pre-delete hook并删除 finalizers ，更新资源对象；
    3. 由于以上两点，需要确保 pre-delete hook是幂等的。

## kuberbuilder 示例

我们来看一个 kubebuilder 官方示例：

```go
func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
    ctx := context.Background()
    log := r.Log.WithValues("cronjob", req.NamespacedName)

    var cronJob batch.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        return ctrl.Result{}, ignoreNotFound(err)
    }

    // 声明 finalizer 字段，类型为字符串
    myFinalizerName := "storage.finalizers.tutorial.kubebuilder.io"

    // 通过检查 DeletionTimestamp 字段是否为0 判断资源是否被删除
    if cronJob.ObjectMeta.DeletionTimestamp.IsZero() {
        // 如果为0 ，则资源未被删除，我们需要检测是否存在 finalizer，如果不存在，则添加，并更新到资源对象中
        if !containsString(cronJob.ObjectMeta.Finalizers, myFinalizerName) {
            cronJob.ObjectMeta.Finalizers = append(cronJob.ObjectMeta.Finalizers, myFinalizerName)
            if err := r.Update(context.Background(), cronJob); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
        // 如果不为 0 ，则对象处于删除中
        if containsString(cronJob.ObjectMeta.Finalizers, myFinalizerName) {
            // 如果存在 finalizer 且与上述声明的 finalizer 匹配，那么执行对应 hook 逻辑
            if err := r.deleteExternalResources(cronJob); err != nil {
                // 如果删除失败，则直接返回对应 err，controller 会自动执行重试逻辑
                return ctrl.Result{}, err
            }

            // 如果对应 hook 执行成功，那么清空 finalizers， k8s 删除对应资源
            cronJob.ObjectMeta.Finalizers = removeString(cronJob.ObjectMeta.Finalizers, myFinalizerName)
            if err := r.Update(context.Background(), cronJob); err != nil {
                return ctrl.Result{}, err
            }
        }

        return ctrl.Result{}, err
    }
}

func (r *Reconciler) deleteExternalResources(cronJob *batch.CronJob) error {
    //
    // 删除 crobJob关联的外部资源逻辑
    //
    // 需要确保实现是幂等的
}

func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}

func removeString(slice []string, s string) (result []string) {
    for _, item := range slice {
        if item == s {
            continue
        }
        result = append(result, item)
    }
    return
}
```

## 总结

在开发 Operator 时，pre-delete hook 是一个很常见的需求，目前只发现了 Finalizers 适合实现这个功能，需要好好掌握。

