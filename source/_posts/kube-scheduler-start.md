---
title: kube-scheduler启动流程
date: 2019-03-01 12:00:44
tags: scheduler
categories: Kubernetes
---

# kube-sheduler 启动流程

## 作用简介

kube-scheduler是k8s中的调度模块，负责调度Pod，pod是k8s最基本的运行单元。再我们剖析代码之前，可以猜测一下k8s是如何工作的。kube-scheduler需要对未被调度的Pod进行Watch，同时也需要对node进行watch，因为pod需要绑定到具体的Node上，当kube-scheduler监测到未被调度的pod，它会取出这个pod，然后依照内部设定的调度算法，选择合适的node，然后通过apiserver写回到etcd，kube-scheduler的工作也就结束了。实际kube-scheduler的代码就是这样工作的。

<!-- more -->

## kube-sheduler feature-gates 介绍

>- APIListChunking=true|false (BETA - default=true)
>- APIResponseCompression=true|false (ALPHA - default=false)

## kube-sheduler 原理介绍

1, Scheduler收集和分析当前Kubernetes集群中所有Minion节点的资源(内存、CPU)负载情况，然后依此分发新建的Pod到Kubernetes集群中可用的节点。
实时监测Kubernetes集群中未分发和已分发的所有运行的Pod。

2, Scheduler也监测Minion节点信息，由于会频繁查找Minion节点，Scheduler会缓存一份最新的信息在本地。

3, 最后，Scheduler在分发Pod到指定的Minion节点后，会把Pod相关的信息Binding写回API Server。

## kube-shedule 服务启动分析

>1. 使用spf13/cobra 包进行命令行封装
>2. 配置leader election 选项则进行 leader 选举功能  LeaderElection 设置true
>3. 运行leader 选举机制acquire方法(maybeReportTransition)
>4. 在主线程运行sched.Run() 并且设置 <-stopCh 主线程为阻塞状态

```go
func Run(c schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	glog.Infof("Version: %+v", version.Get())

	// Apply algorithms based on feature gates.
	// TODO: make configurable?
	algorithmprovider.ApplyFeatureGates()

	// Configz registration.
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(c.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	// Build a scheduler config from the provided algorithm source.
	schedulerConfig, err := NewSchedulerConfig(c)
	if err != nil {
		return err
	}

	// Create the scheduler.
	sched := scheduler.NewFromConfig(schedulerConfig)

	// Prepare the event broadcaster.
	if c.Broadcaster != nil && c.EventClient != nil {
		c.Broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: c.EventClient.Events("")})
	}

	// Start up the healthz server.
	if c.InsecureServing != nil {
		separateMetrics := c.InsecureMetricsServing != nil
		handler := buildHandlerChain(newHealthzHandler(&c.ComponentConfig, separateMetrics), nil, nil)
		if err := c.InsecureServing.Serve(handler, 0, stopCh); err != nil {
			return fmt.Errorf("failed to start healthz server: %v", err)
		}
	}
	if c.InsecureMetricsServing != nil {
		handler := buildHandlerChain(newMetricsHandler(&c.ComponentConfig), nil, nil)
		if err := c.InsecureMetricsServing.Serve(handler, 0, stopCh); err != nil {
			return fmt.Errorf("failed to start metrics server: %v", err)
		}
	}
	if c.SecureServing != nil {
		handler := buildHandlerChain(newHealthzHandler(&c.ComponentConfig, false), c.Authentication.Authenticator, c.Authorization.Authorizer)
		if err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			// fail early for secure handlers, removing the old error loop from above
			return fmt.Errorf("failed to start healthz server: %v", err)
		}
	}

	// Start all informers.
	go c.PodInformer.Informer().Run(stopCh)
	c.InformerFactory.Start(stopCh)

	// Wait for all caches to sync before scheduling.
	c.InformerFactory.WaitForCacheSync(stopCh)
	controller.WaitForCacheSync("scheduler", stopCh, c.PodInformer.Informer().HasSynced)

	// Prepare a reusable run function.
	run := func(stopCh <-chan struct{}) {
		sched.Run()
		<-stopCh
	}

	// If leader election is enabled, run via LeaderElector until done and exit.
	if c.LeaderElection != nil {
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

		leaderElector.Run()

		return fmt.Errorf("lost lease")
	}

	// Leader election is disabled, so run inline until done.
	run(stopCh)
	return fmt.Errorf("finished without leader elect")
}
```

## kube-sheduler 逻辑代码分析

```go
// Run begins watching and scheduling. It waits for cache to be synced, then starts a goroutine and returns immediately.
func (sched *Scheduler) Run() {
	if !sched.config.WaitForCacheSync() {
		return
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
		go sched.config.VolumeBinder.Run(sched.bindVolumesWorker, sched.config.StopEverything)
	}

	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```

## kube-sheduler 逻辑调度流程

```go
// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	trace := utiltrace.New(fmt.Sprintf("Scheduling %s/%s", pod.Namespace, pod.Name))
	defer trace.LogIfLong(100 * time.Millisecond)

        // 检查pod 中需要pvc 是否存在
	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return "", err
	}

    // 获取所有node节点
	nodes, err := nodeLister.List()
	if err != nil {
		return "", err
	}
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// Used for all fit and priority funcs.
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	if err != nil {
		return "", err
	}

	trace.Step("Computing predicates")
	startPredicateEvalTime := time.Now()
	filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
	if err != nil {
		return "", err
	}

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			NumAllNodes:      len(nodes),
			FailedPredicates: failedPredicateMap,
		}
	}
	metrics.SchedulingAlgorithmPredicateEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPredicateEvalTime))
	metrics.SchedulingLatency.WithLabelValues(metrics.PredicateEvaluation).Observe(metrics.SinceInSeconds(startPredicateEvalTime))

	trace.Step("Prioritizing")
	startPriorityEvalTime := time.Now()
	// When only one node after predicate, just use it.
	if len(filteredNodes) == 1 {
		metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
		return filteredNodes[0].Name, nil
	}

	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	if err != nil {
		return "", err
	}
	metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
	metrics.SchedulingLatency.WithLabelValues(metrics.PriorityEvaluation).Observe(metrics.SinceInSeconds(startPriorityEvalTime))

	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```

```go
// Filters the nodes to find the ones that fit based on the given predicate functions
// Each node is passed through the predicate functions to determine if it is a fit
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}

	if len(g.predicates) == 0 {
		filtered = nodes
	} else {
		allNodes := int32(g.cache.NodeTree().NumNodes)
		numNodesToFind := g.numFeasibleNodesToFind(allNodes)

		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, numNodesToFind)
		errs := errors.MessageCountMap{}
		var (
			predicateResultLock sync.Mutex
			filteredLen         int32
			equivClass          *equivalence.Class
		)

		ctx, cancel := context.WithCancel(context.Background())

		// We can use the same metadata producer for all nodes.
		meta := g.predicateMetaProducer(pod, g.cachedNodeInfoMap)

		if g.equivalenceCache != nil {
			// getEquivalenceClassInfo will return immediately if no equivalence pod found
			equivClass = equivalence.NewClass(pod)
		}

		checkNode := func(i int) {
			var nodeCache *equivalence.NodeCache
			nodeName := g.cache.NodeTree().Next()
			if g.equivalenceCache != nil {
				nodeCache, _ = g.equivalenceCache.GetNodeCache(nodeName)
			}
			fits, failedPredicates, err := podFitsOnNode(
				pod,
				meta,
				g.cachedNodeInfoMap[nodeName],
				g.predicates,
				g.cache,
				nodeCache,
				g.schedulingQueue,
				g.alwaysCheckAllPredicates,
				equivClass,
			)
			if err != nil {
				predicateResultLock.Lock()
				errs[err.Error()]++
				predicateResultLock.Unlock()
				return
			}
			if fits {
				length := atomic.AddInt32(&filteredLen, 1)
				if length > numNodesToFind {
					cancel()
					atomic.AddInt32(&filteredLen, -1)
				} else {
					filtered[length-1] = g.cachedNodeInfoMap[nodeName].Node()
				}
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}

		// Stops searching for more nodes once the configured number of feasible nodes
		// are found.
		workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode) //开启最多16，最少node数量个线程并发调度node资源

		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}

	if len(filtered) > 0 && len(g.extenders) != 0 {
		for _, extender := range g.extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			filteredList, failedMap, err := extender.Filter(pod, filtered, g.cachedNodeInfoMap)
			if err != nil {
				if extender.IsIgnorable() {
					glog.Warningf("Skipping extender %v as it returned error %v and has ignorable flag set",
						extender, err)
					continue
				} else {
					return []*v1.Node{}, FailedPredicateMap{}, err
				}
			}

			for failedNodeName, failedMsg := range failedMap {
				if _, found := failedPredicateMap[failedNodeName]; !found {
					failedPredicateMap[failedNodeName] = []algorithm.PredicateFailureReason{}
				}
				failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
			}
			filtered = filteredList
			if len(filtered) == 0 {
				break
			}
		}
	}
	return filtered, failedPredicateMap, nil
}
```
