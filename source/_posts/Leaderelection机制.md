---
title: Leaderelection机制
categories: Kubernetes
sage: false
date: 2020-07-03 14:14:18
tags: k8s
---

## 前言

最近在开发的ingress-controller组件，副本是有状态服务。每个副本将收到的ingress事件进行解析，然后与slb同步。如果同时多个副本运行，势必会造成对slb访问的混乱。因此，同一时刻，只能有一个副本真正在工作。但是，还需要多副本部署方式来保证高可用。
为了解决这个问题，本组件参考kube-scheduler、kube-controller-manager等组件的实现方式，也利用到client-go/tools/leaderelection的选主机制，保证只有leader处于工作状态，并定时进行leader的重新选举或续租。当leader挂掉之后，从其他节点选举新的leader以保证组件正常工作。

<!-- more -->

本文以ingress-controller组件为例，讲述如何使用leaderelection。并深入分析它的实现原理。

## 使用

### 1、首先创建leaderElectionClient

```go
config, err := buildConfig(conf.KubeConfigFile)
if err != nil {
    klog.Fatal(err)
}
leaderElectionClient := kubernetes.NewForConfigOrDie(config)

func buildConfig(kubeconfig string) (*rest.Config, error) {
if kubeconfig != "" {
    cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return nil, err
    }
    return cfg, nil
}

cfg, err := rest.InClusterConfig()
if err != nil {
    return nil, err
}
return cfg, nil
}
```

### 2、创建event recorder，记录选举产生的事件

```go
eventBroadcaster := record.NewBroadcaster()
eventBroadcaster.StartLogging(klog.Infof)
eventBroadcaster.StartRecordingToSink(&clientcorev1.EventSinkImpl{
    Interface: kubeClient.CoreV1().Events(conf.Namespace),
})
recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{
    Component: "ksyun-ingress-controller",
})
```

### 3、选举为leader后，所做的工作，即为run函数中的内容

```go
ctl := controller.NewKSyunIngressController(conf)
run := func(ctx context.Context) {
    ctl.Start()    //watch ingress相关事件，更新slb
    panic("unreachable")
}
```

### 4、设置节点标识、资源锁、ctx
```go
// Identity used to distinguish between multiple ingress controller instances
id, err := os.Hostname()    //用于标识当前节点，一般用当前的主机名
if err != nil {
    klog.Fatalf("error getting hostname: %+v", err)
}

// Lock required for leader election
rl := resourcelock.EndpointsLock{//resourcelock的类型为endpoint，即选举信息存放在endpoint中
    EndpointsMeta: metav1.ObjectMeta{
        Namespace: "kube-system",
        Name:      "ksyun-ingress-controller",
    },
    Client: leaderElectionClient.CoreV1(),
    LockConfig: resourcelock.ResourceLockConfig{
        Identity:      id,
        EventRecorder: recorder,
    },
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```
### 5、开始leader选举loop，成为leader的节点执行run操作
```go
// Try and become the leader and start ingress controller manager loops

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock: &rl,   //资源锁
    LeaseDuration: DefaultLeaseDuration,  //租约有效时间
    RenewDeadline: DefaultRenewDeadline,   //更新租约的时间
    RetryPeriod: DefaultRetryPeriod, //非leader重新尝试选举的事件间隔
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            run(ctx)     //变为leader执行的业务代码
        },
        OnStoppedLeading: func() {  //节点退出
            klog.Fatalf("leaderelection lost")
        },
    },
})
```

## 原理

利用通过Kubernetes中 configmap ， endpoints 或者 lease 资源实现一个分布式锁，抢(acqure)到锁的节点成为leader，并且定期更新（renew）。其他进程也在不断的尝试进行抢占，抢占不到则继续等待下次循环。当leader节点挂掉之后，租约到期，其他节点就成为新的leader。

LeaderElectionConfig.lock 支持保存在以下三种资源锁：configmap、endpoint 、lease 
包中还提供了一个 multilock ，即可以进行选择两种，当其中一种保存失败时，选择第二张可以在interface.go中看到：

```go
	switch lockType {
	case EndpointsResourceLock://保存在endpoints
		return endpointsLock, nil
	case ConfigMapsResourceLock://保存在configmaps
		return configmapLock, nil
	case LeasesResourceLock://保存在leases
		return leaseLock, nil
	case EndpointsLeasesResourceLock://优先尝试保存在endpoint失败时保存在lease
		return &MultiLock{
			Primary:   endpointsLock,
			Secondary: leaseLock,
		}, nil
	case ConfigMapsLeasesResourceLock://优先尝试保存在configmap，失败时保存在lease
		return &MultiLock{
			Primary:   configmapLock,
			Secondary: leaseLock,
		}, nil
	default:
		return nil, fmt.Errorf("Invalid lock-type %s", lockType)
	}
```
在本组件中采用的是endpoint资源锁，可以通过查看endpoint的yaml，检查选举信息

我们重点看上一节中的第5部分：leaderelection.RunOrDie()入口在client-go/tools/leaderelection/leaderelection.go

```go
// RunOrDie starts a client with the provided config or panics if the config
// fails to validate. RunOrDie blocks until leader election loop is
// stopped by ctx or it has stopped holding the leader lease
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
    le, err := NewLeaderElector(lec)
    if err != nil {
        panic(err)
    }
    if lec.WatchDog != nil {
        lec.WatchDog.SetLeaderElection(le)
    }
    le.Run(ctx)  //负责获取锁和更新锁
}

// Run starts the leader election loop. Run will not return
// before leader election loop is stopped by ctx or it has
// stopped holding the leader lease
func (le *LeaderElector) Run(ctx context.Context) {
    defer runtime.HandleCrash()
    defer func() {
        le.config.Callbacks.OnStoppedLeading()
    }()
    if !le.acquire(ctx) { //不停地去竞争锁，如果成功，则执行后续的操作，否则一直尝试
        return // ctx signalled done 
    }
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    go le.config.Callbacks.OnStartedLeading(ctx) //获取锁成功，变为leader，执行业务代码
    le.renew(ctx) //循环进行租约的更新，保证锁一直被当前节点hold。 
}
```

acquire和renew中都使用了"k8s.io/apimachinery/pkg/util/wait"中的wait包进行循环操作。当acquire中某一次循环执行成功时，会退出获取循环，进行接下来的操作；
如果不成功，则一直循环获取，直到退出竞争。而renew是进行更新循环，一次循环成功，会继续循环，不行的续约。如果某次循环失败，则退出续约操作，即变更leader
两者内部都是调用tryAcquireOrRenew()

```go
func (le *LeaderElector) tryAcquireOrRenew() bool {
    now := metav1.Now()
    //锁资源对象内容
    leaderElectionRecord := rl.LeaderElectionRecord{
        HolderIdentity: le.config.Lock.Identity(),//唯一标识
        LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
        RenewTime: now,
        AcquireTime: now,
    }

    // 1. obtain or create the ElectionRecord
    // 第一步：从k8s资源中获取原有的锁
    oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get()
    if err != nil {
        if !errors.IsNotFound(err) {
            klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
            return false
        }
        //资源对象不存在，进行锁资源创建
        if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
            klog.Errorf("error initially creating leader election record: %v", err)
            return false
        }
        le.observedRecord = leaderElectionRecord
        le.observedTime = le.clock.Now()
        return true
    }

    // 第二步，对比存储在k8s中的锁资源与上一次获取的锁资源是否一致
    if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
        le.observedRecord = *oldLeaderElectionRecord
        le.observedRawRecord = oldLeaderElectionRawRecord
        le.observedTime = le.clock.Now()
    }

    //判断持有的锁是否到期以及是否被自己持有
    if len(oldLeaderElectionRecord.HolderIdentity) > 0 && le.observedTime.Add(le.config.LeaseDuration).After(now.Time) && !le.IsLeader() {
        klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
        return false
    }

    //第三步：自己现在是leader，但是分两组情况，上一次也是leader和首次变为leader
    if le.IsLeader() {
        //自己本身就是leader则不需要更新AcquireTime和LeaderTransitions
        leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
    } else {
        //首次自己变为leader则更新leader的更换次数
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
    }

    //更新锁资源，这里如果在 Get 和 Update 之间有变化，将会更新失败
    if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
        klog.Errorf("Failed to update lock: %v", err)
        return false
    }

    le.observedRecord = leaderElectionRecord
    le.observedTime = le.clock.Now()
    return true
}
```