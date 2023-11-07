---
title: leader选举源码分析
date: 2019-01-08 10:08:07
tags: scheduler
categories: Kubernetes
---

# kube-sheduler leader 选举代码分析

> 所使用的包clinet-go 模块中的tools/leaderelection 包

```bash
// If leader election is enabled, run via LeaderElector until done and exit.
	if c.LeaderElection != nil {
        // 设置callbacks 方法started/stopped
		c.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				utilruntime.HandleError(fmt.Errorf("lost master"))
			},
		}

		leaderElector, err := leaderelection.NewLeaderElector(*c.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

        // 启动选举方法
		leaderElector.Run()

		return fmt.Errorf("lost lease")
	}
```

<!-- more -->

```bash
// 使用client-go 模块中的tools/leaderelection包
// Run starts the leader election loop
func (le *LeaderElector) Run() {
    // 设置函数返回之前执行的callbacks方法
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
    // 获取和选举leader信息 如果不是leader 则一直循环选举等待
	le.acquire()
	stop := make(chan struct{})
    // 运行scheduler 的逻辑代码 stop 管道设置为阻塞状态
	go le.config.Callbacks.OnStartedLeading(stop)
    // 选举新的leader
	le.renew()
	close(stop)
}
```

```bash
// acquire loops calling tryAcquireOrRenew and returns immediately when tryAcquireOrRenew succeeds.
// 周期性选举leader，直到自己成为leader后才推出循环
func (le *LeaderElector) acquire() {
	stop := make(chan struct{})
    // 输出命名空间和名称eg: kube-system/kube-scheduler
	desc := le.config.Lock.Describe()
	glog.Infof("attempting to acquire leader lease  %v...", desc)
    // 调用apimachinery 模块中的wait包中JitterUntil循环执行函数
	wait.JitterUntil(func() {
		succeeded := le.tryAcquireOrRenew()
		le.maybeReportTransition()
        // 
		if !succeeded {
			glog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("became leader")
		glog.Infof("successfully acquired lease %v", desc)
        // 
		close(stop)
	}, le.config.RetryPeriod, JitterFactor, true, stop)
}
```

```bash
// tryAcquireOrRenew tries to acquire a leader lease if it is not already acquired,
// else it tries to renew the lease if it has already been acquired. Returns true
// on success else returns false.
// 获取leader信息是否是本机
func (le *LeaderElector) tryAcquireOrRenew() bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1. obtain or create the ElectionRecord
	oldLeaderElectionRecord, err := le.config.Lock.Get()
	if err != nil {
		if !errors.IsNotFound(err) {
			glog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}
		if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
			glog.Errorf("error initially creating leader election record: %v", err)
			return false
		}
		le.observedRecord = leaderElectionRecord
		le.observedTime = time.Now()
		return true
	}

	// 2. Record obtained, check the Identity & Time
	if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
		le.observedRecord = *oldLeaderElectionRecord
		le.observedTime = time.Now()
	}
	if le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		oldLeaderElectionRecord.HolderIdentity != le.config.Lock.Identity() {
		glog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	// 3. We're going to try to update. The leaderElectionRecord is set to it's default
	// here. Let's correct it before updating.
	if oldLeaderElectionRecord.HolderIdentity == le.config.Lock.Identity() {
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// update the lock itself
	if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
		glog.Errorf("Failed to update lock: %v", err)
		return false
	}
	le.observedRecord = leaderElectionRecord
	le.observedTime = time.Now()
	return true
}
```

```bash
// renew loops calling tryAcquireOrRenew and returns immediately when tryAcquireOrRenew fails.
// 间歇性获取leader信息是否是本机
func (le *LeaderElector) renew() {
	stop := make(chan struct{})
	wait.Until(func() {
		err := wait.Poll(le.config.RetryPeriod, le.config.RenewDeadline, func() (bool, error) {
			return le.tryAcquireOrRenew(), nil
		})
		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		if err == nil {
			glog.V(4).Infof("successfully renewed lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("stopped leading")
		glog.Infof("failed to renew lease %v: %v", desc, err)
		close(stop)
	}, 0, stop)
}
```