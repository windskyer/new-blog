---
title: AutoScaler源码梳理
date: 2019-03-06 14:43:20
tags: k8s
categories: [Kubernetes]
---

# Cluster AutoScaler源码梳理

**功能总结:**

>1. 删除nodegroup中15分钟没有注册到kube的node
>2. 获取未调度的Pod列表，计算需要资源（limit之内）
>3. 获取comingnode（在nodegroup内，但是未注册kube集群，启动中）+node模版，计算（base，binpacking）的需要的node数
>4. 根据expander(random,most-pods,price,least-waste)进行scaleup
>5. 更新nodegroup的currentsize

<!-- more -->

**cluster ca 入口函数:**

```go
func main() {
  
   leaderElection := defaultLeaderElectionConfiguration()
   //默认进行选举
   leaderElection.LeaderElect = true
  
   bindFlags(&leaderElection, pflag.CommandLine)
  
   kube_flag.InitFlags()
  
   //初始化健康检查 超过15分钟失败
   healthCheck := metrics.NewHealthCheck(*maxInactivityTimeFlag, *maxFailingTimeFlag)
  
   glog.V(1).Infof("Cluster Autoscaler %s", ClusterAutoscalerVersion)
  
   correctEstimator := false
   //评估算法 basic binpacking
   for _, availableEstimator := range estimator.AvailableEstimators {
      if *estimatorFlag == availableEstimator {
         correctEstimator = true
      }
   }
   if !correctEstimator {
      glog.Fatalf("Unrecognized estimator: %v", *estimatorFlag)
   }
  
   //注册监控
   //注册健康检查
   go func() {
      http.Handle("/metrics", prometheus.Handler())
      http.Handle("/health-check", healthCheck)
      err := http.ListenAndServe(*address, nil)
      glog.Fatalf("Failed to start metrics: %v", err)
   }()
  
   if !leaderElection.LeaderElect {
      run(healthCheck)
   } else {
      //默认
      id, err := os.Hostname()
      if err != nil {
         glog.Fatalf("Unable to get hostname: %v", err)
      }
  
      kubeClient := createKubeClient(getKubeConfig())
  
      // Validate that the client is ok.
      _, err = kubeClient.CoreV1().Nodes().List(metav1.ListOptions{})
      if err != nil {
         glog.Fatalf("Failed to get nodes from apiserver: %v", err)
      }
  
      lock, err := resourcelock.New(
         leaderElection.ResourceLock,
         *namespace,
         "cluster-autoscaler",
         kubeClient.CoreV1(),
         resourcelock.ResourceLockConfig{
            Identity:      id,
            EventRecorder: kube_util.CreateEventRecorder(kubeClient),
         },
      )
      if err != nil {
         glog.Fatalf("Unable to create leader election lock: %v", err)
      }
  
      kube_leaderelection.RunOrDie(kube_leaderelection.LeaderElectionConfig{
         Lock:          lock,
         LeaseDuration: leaderElection.LeaseDuration.Duration,
         RenewDeadline: leaderElection.RenewDeadline.Duration,
         RetryPeriod:   leaderElection.RetryPeriod.Duration,
         Callbacks: kube_leaderelection.LeaderCallbacks{
            OnStartedLeading: func(_ <-chan struct{}) {
               // Since we are committing a suicide after losing
               // mastership, we can safely ignore the argument.
               //run 入口
               run(healthCheck)
            },
            OnStoppedLeading: func() {
               glog.Fatalf("lost master")
            },
         },
      })
   }
}
```

**run 方法:**

```go
func run(healthCheck *metrics.HealthCheck) {
   metrics.RegisterAll()
   kubeConfig := getKubeConfig()
   kubeClient := createKubeClient(kubeConfig)
   kubeEventRecorder := kube_util.CreateEventRecorder(kubeClient)
   autoscalingOptions := createAutoscalingOptions()
   metrics.UpdateNapEnabled(autoscalingOptions.NodeAutoprovisioningEnabled)
   predicateCheckerStopChannel := make(chan struct{})
   predicateChecker, err := simulator.NewPredicateChecker(kubeClient, predicateCheckerStopChannel)
   if err != nil {
      glog.Fatalf("Failed to create predicate checker: %v", err)
   }
   listerRegistryStopChannel := make(chan struct{})
   listerRegistry := kube_util.NewListerRegistryWithDefaultListers(kubeClient, listerRegistryStopChannel)
 
   opts := core.AutoscalerOptions{
      AutoscalingOptions: autoscalingOptions,
      PredicateChecker:   predicateChecker,
      KubeClient:         kubeClient,
      KubeEventRecorder:  kubeEventRecorder,
      ListerRegistry:     listerRegistry,
   }
   //初始化autoscaler
   autoscaler, err := core.NewAutoscaler(opts)
   if err != nil {
      glog.Fatalf("Failed to create autoscaler: %v", err)
   }
   registerSignalHandlers(autoscaler)
   healthCheck.StartMonitoring()
 
   for {
      select {
      //默认15s CA执行一次
      case <-time.After(*scanInterval):
         {
            loopStart := time.Now()
            metrics.UpdateLastTime(metrics.Main, loopStart)
            //更新健康信息
            healthCheck.UpdateLastActivity(loopStart)
            //执行CA
            err := autoscaler.RunOnce(loopStart)
            if err != nil && err.Type() != errors.TransientError {
               metrics.RegisterError(err)
            } else {
               healthCheck.UpdateLastSuccessfulRun(time.Now())
            }
            //更新metrics
            metrics.UpdateDurationFromStart(metrics.Main, loopStart)
         }
      }
   }
}
```

**New Autoscaler:**

```go
func (a *StaticAutoscaler) RunOnce(currentTime time.Time) errors.AutoscalerError {
   a.cleanUpIfRequired()
   //kubernetes Lister
   unschedulablePodLister := a.UnschedulablePodLister()
   scheduledPodLister := a.ScheduledPodLister()
   pdbLister := a.PodDisruptionBudgetLister()
   scaleDown := a.scaleDown
   //context
   autoscalingContext := a.AutoscalingContext
 
   glog.V(4).Info("Starting main loop")
 
   stateUpdateStart := time.Now()
   //获取全部node
   allNodes, readyNodes, typedErr := a.obtainNodeLists()
   if typedErr != nil {
      return typedErr
   }
   if a.actOnEmptyCluster(allNodes, readyNodes, currentTime) {
      return nil
   }
   //更新cluster status
   typedErr = a.updateClusterState(allNodes, currentTime)
   if typedErr != nil {
      return typedErr
   }
   metrics.UpdateDurationFromStart(metrics.UpdateState, stateUpdateStart)
 
   // Update status information when the loop is done (regardless of reason)
   defer func() {
      if autoscalingContext.WriteStatusConfigMap {
         status := a.clusterStateRegistry.GetStatus(currentTime)
         utils.WriteStatusConfigMap(autoscalingContext.ClientSet, autoscalingContext.ConfigNamespace,
            status.GetReadableString(), a.AutoscalingContext.LogRecorder)
      }
   }()
   // Check if there are any nodes that failed to register in Kubernetes
   // 获取unregistered的node（node在CA group中，但是没有注册在kube中）
   unregisteredNodes := a.clusterStateRegistry.GetUnregisteredNodes()
   // 删除unregistered的node
   if len(unregisteredNodes) > 0 {
      glog.V(0).Infof("%d unregistered nodes present", len(unregisteredNodes))
      removedAny, err := removeOldUnregisteredNodes(unregisteredNodes, autoscalingContext, currentTime, autoscalingContext.LogRecorder)
      // There was a problem with removing unregistered nodes. Retry in the next loop.
      if err != nil {
         if removedAny {
            glog.Warningf("Some unregistered nodes were removed, but got error: %v", err)
         } else {
            glog.Errorf("Failed to remove unregistered nodes: %v", err)
 
         }
         return errors.ToAutoscalerError(errors.CloudProviderError, err)
      }
      // Some nodes were removed. Let's skip this iteration, the next one should be better.
      if removedAny {
         glog.V(0).Infof("Some unregistered nodes were removed, skipping iteration")
         return nil
      }
   }
   //集群不健康 csr.totalReadiness.Unready + csr.totalReadiness.LongNotStarted + csr.totalReadiness.LongUnregistered > csr.config.OkTotalUnreadyCount
   if !a.clusterStateRegistry.IsClusterHealthy() {
      glog.Warning("Cluster is not ready for autoscaling")
      scaleDown.CleanUpUnneededNodes()
      autoscalingContext.LogRecorder.Eventf(apiv1.EventTypeWarning, "ClusterUnhealthy", "Cluster is unhealthy")
      return nil
   }
 
   // Check if there has been a constant difference between the number of nodes in k8s and
   // the number of nodes on the cloud provider side.
   // TODO: andrewskim - add protection for ready AWS nodes.
   fixedSomething, err := fixNodeGroupSize(autoscalingContext, a.clusterStateRegistry, currentTime)
   if err != nil {
      glog.Errorf("Failed to fix node group sizes: %v", err)
      return errors.ToAutoscalerError(errors.CloudProviderError, err)
   }
   if fixedSomething {
      glog.V(0).Infof("Some node group target size was fixed, skipping the iteration")
      return nil
   }
 
   metrics.UpdateLastTime(metrics.Autoscaling, time.Now())
 
   //获取未调度pod列表
   allUnschedulablePods, err := unschedulablePodLister.List()
   if err != nil {
      glog.Errorf("Failed to list unscheduled pods: %v", err)
      return errors.ToAutoscalerError(errors.ApiCallError, err)
   }
   //更新metrics
   metrics.UpdateUnschedulablePodsCount(len(allUnschedulablePods))
 
   allScheduled, err := scheduledPodLister.List()
   if err != nil {
      glog.Errorf("Failed to list scheduled pods: %v", err)
      return errors.ToAutoscalerError(errors.ApiCallError, err)
   }
 
   allUnschedulablePods, allScheduled, err = a.processors.PodListProcessor.Process(a.AutoscalingContext, allUnschedulablePods, allScheduled, allNodes)
   if err != nil {
      glog.Errorf("Failed to process pod list: %v", err)
      return errors.ToAutoscalerError(errors.InternalError, err)
   }
 
   ConfigurePredicateCheckerForLoop(allUnschedulablePods, allScheduled, a.PredicateChecker)
 
   // We need to check whether pods marked as unschedulable are actually unschedulable.
   // It's likely we added a new node and the scheduler just haven't managed to put the
   // pod on in yet. In this situation we don't want to trigger another scale-up.
   //
   // It's also important to prevent uncontrollable cluster growth if CA's simulated
   // scheduler differs in opinion with real scheduler. Example of such situation:
   // - CA and Scheduler has slightly different configuration
   // - Scheduler can't schedule a pod and marks it as unschedulable
   // - CA added a node which should help the pod
   // - Scheduler doesn't schedule the pod on the new node
   //   because according to it logic it doesn't fit there
   // - CA see the pod is still unschedulable, so it adds another node to help it
   //
   // With the check enabled the last point won't happen because CA will ignore a pod
   // which is supposed to schedule on an existing node.
   scaleDownForbidden := false
 
   unschedulablePodsWithoutTPUs := tpu.ClearTPURequests(allUnschedulablePods)
 
   // Some unschedulable pods can be waiting for lower priority pods preemption so they have nominated node to run.
   // Such pods don't require scale up but should be considered during scale down.
   unschedulablePods, unschedulableWaitingForLowerPriorityPreemption := FilterOutExpendableAndSplit(unschedulablePodsWithoutTPUs, a.ExpendablePodsPriorityCutoff)
 
   glog.V(4).Infof("Filtering out schedulables")
   filterOutSchedulableStart := time.Now()
 
   unschedulablePodsToHelp := FilterOutSchedulable(unschedulablePods, readyNodes, allScheduled,
      unschedulableWaitingForLowerPriorityPreemption, a.PredicateChecker, a.ExpendablePodsPriorityCutoff)
   metrics.UpdateDurationFromStart(metrics.FilterOutSchedulable, filterOutSchedulableStart)
 
   if len(unschedulablePodsToHelp) != len(unschedulablePods) {
      glog.V(2).Info("Schedulable pods present")
      scaleDownForbidden = true
   } else {
      glog.V(4).Info("No schedulable pods")
   }
 
   if len(unschedulablePodsToHelp) == 0 {
      glog.V(1).Info("No unschedulable pods")
   } else if a.MaxNodesTotal > 0 && len(readyNodes) >= a.MaxNodesTotal {
      glog.V(1).Info("Max total nodes in cluster reached")
   } else if allPodsAreNew(unschedulablePodsToHelp, currentTime) {
      // The assumption here is that these pods have been created very recently and probably there
      // is more pods to come. In theory we could check the newest pod time but then if pod were created
      // slowly but at the pace of 1 every 2 seconds then no scale up would be triggered for long time.
      // We also want to skip a real scale down (just like if the pods were handled).
      scaleDownForbidden = true
      glog.V(1).Info("Unschedulable pods are very new, waiting one iteration for more")
   } else {
      daemonsets, err := a.ListerRegistry.DaemonSetLister().List()
      if err != nil {
         glog.Errorf("Failed to get daemonset list")
         return errors.ToAutoscalerError(errors.ApiCallError, err)
      }
 
      scaleUpStart := time.Now()
      metrics.UpdateLastTime(metrics.ScaleUp, scaleUpStart)
 
      scaleUpStatus, typedErr := ScaleUp(autoscalingContext, a.processors, a.clusterStateRegistry, unschedulablePodsToHelp, readyNodes, daemonsets)
 
      metrics.UpdateDurationFromStart(metrics.ScaleUp, scaleUpStart)
 
      if typedErr != nil {
         glog.Errorf("Failed to scale up: %v", typedErr)
         return typedErr
      }
      if a.processors != nil && a.processors.ScaleUpStatusProcessor != nil {
         a.processors.ScaleUpStatusProcessor.Process(autoscalingContext, scaleUpStatus)
      }
      if scaleUpStatus.ScaledUp {
         a.lastScaleUpTime = currentTime
         // No scale down in this iteration.
         return nil
      }
   }
 
   if a.ScaleDownEnabled {
      pdbs, err := pdbLister.List()
      if err != nil {
         glog.Errorf("Failed to list pod disruption budgets: %v", err)
         return errors.ToAutoscalerError(errors.ApiCallError, err)
      }
 
      unneededStart := time.Now()
 
      glog.V(4).Infof("Calculating unneeded nodes")
 
      scaleDown.CleanUp(currentTime)
      potentiallyUnneeded := getPotentiallyUnneededNodes(autoscalingContext, allNodes)
 
      typedErr := scaleDown.UpdateUnneededNodes(allNodes, potentiallyUnneeded, append(allScheduled, unschedulableWaitingForLowerPriorityPreemption...), currentTime, pdbs)
      if typedErr != nil {
         glog.Errorf("Failed to scale down: %v", typedErr)
         return typedErr
      }
 
      metrics.UpdateDurationFromStart(metrics.FindUnneeded, unneededStart)
 
      if glog.V(4) {
         for key, val := range scaleDown.unneededNodes {
            glog.Infof("%s is unneeded since %s duration %s", key, val.String(), currentTime.Sub(val).String())
         }
      }
 
      // In dry run only utilization is updated
      calculateUnneededOnly := scaleDownForbidden ||
         a.lastScaleUpTime.Add(a.ScaleDownDelayAfterAdd).After(currentTime) ||
         a.lastScaleDownFailTime.Add(a.ScaleDownDelayAfterFailure).After(currentTime) ||
         a.lastScaleDownDeleteTime.Add(a.ScaleDownDelayAfterDelete).After(currentTime) ||
         scaleDown.nodeDeleteStatus.IsDeleteInProgress()
 
      glog.V(4).Infof("Scale down status: unneededOnly=%v lastScaleUpTime=%s "+
         "lastScaleDownDeleteTime=%v lastScaleDownFailTime=%s scaleDownForbidden=%v isDeleteInProgress=%v",
         calculateUnneededOnly, a.lastScaleUpTime, a.lastScaleDownDeleteTime, a.lastScaleDownFailTime,
         scaleDownForbidden, scaleDown.nodeDeleteStatus.IsDeleteInProgress())
 
      if !calculateUnneededOnly {
         glog.V(4).Infof("Starting scale down")
 
         // We want to delete unneeded Node Groups only if there was no recent scale up,
         // and there is no current delete in progress and there was no recent errors.
         a.processors.NodeGroupManager.RemoveUnneededNodeGroups(autoscalingContext)
 
         scaleDownStart := time.Now()
         metrics.UpdateLastTime(metrics.ScaleDown, scaleDownStart)
         result, typedErr := scaleDown.TryToScaleDown(allNodes, allScheduled, pdbs, currentTime)
         metrics.UpdateDurationFromStart(metrics.ScaleDown, scaleDownStart)
 
         if typedErr != nil {
            glog.Errorf("Failed to scale down: %v", err)
            a.lastScaleDownFailTime = currentTime
            return typedErr
         }
         if result == ScaleDownNodeDeleted {
            a.lastScaleDownDeleteTime = currentTime
         }
      }
   }
   return nil
}
```

**Autoscaler ScaleUp:**

```go
func ScaleUp(context *context.AutoscalingContext, processors *ca_processors.AutoscalingProcessors, clusterStateRegistry *clusterstate.ClusterStateRegistry, unschedulablePods []*apiv1.Pod,
   nodes []*apiv1.Node, daemonSets []*extensionsv1.DaemonSet) (*status.ScaleUpStatus, errors.AutoscalerError) {
   // From now on we only care about unschedulable pods that were marked after the newest
   // node became available for the scheduler.
   glog.V(0).Info("kce cloud provider begin scale up ***********************")
 
   //未调度的pod集合
   if len(unschedulablePods) == 0 {
      glog.V(1).Info("No unschedulable pods")
      return &status.ScaleUpStatus{ScaledUp: false}, nil
   }
 
   now := time.Now()
 
   loggingQuota := glogx.PodsLoggingQuota()
 
   podsRemainUnschedulable := make(map[*apiv1.Pod]bool)
 
   for _, pod := range unschedulablePods {
      glogx.V(1).UpTo(loggingQuota).Infof("Pod %s/%s is unschedulable", pod.Namespace, pod.Name)
      podsRemainUnschedulable[pod] = true
   }
   glogx.V(1).Over(loggingQuota).Infof("%v other pods are also unschedulable", -loggingQuota.Left())
 
   //获取集群中全部Node与其对应的NodeGroup
   nodeInfos, err := GetNodeInfosForGroups(nodes, context.CloudProvider, context.ClientSet,
      daemonSets, context.PredicateChecker)
 
   //todo
   for k,v := range nodeInfos{
      glog.V(0).Infof("kce get nodegroup by all node， nodegroup: %s , nodeinfo: %v", k,v)
   }
 
   if err != nil {
      return nil, err.AddPrefix("failed to build node infos for node groups: ")
   }
   //NodeGroup外的Nodes
   nodesFromNotAutoscaledGroups, err := FilterOutNodesFromNotAutoscaledGroups(nodes, context.CloudProvider)
 
   for notGroupNode := range nodesFromNotAutoscaledGroups{
      glog.V(0).Infof("kce node not in FilterOutNodesFromNotAutoscaledGroups: node : %v",notGroupNode)
   }
 
   if err != nil {
      return nil, err.AddPrefix("failed to filter out nodes which are from not autoscaled groups: ")
   }
 
   nodeGroups := context.CloudProvider.NodeGroups()
 
   for ng := range nodeGroups{
      glog.V(0).Infof("kce provider autoscale node group from yaml : %v",ng)
   }
 
   //资源限额
   resourceLimiter, errCP := context.CloudProvider.GetResourceLimiter()
 
   glog.V(0).Infof("kce  autoscale resource limiter : %v",resourceLimiter)
   if errCP != nil {
      return nil, errors.ToAutoscalerError(
         errors.CloudProviderError,
         errCP)
   }
   //调度之后的剩余Recourse
   scaleUpResourcesLeft, errLimits := computeScaleUpResourcesLeftLimits(nodeGroups, nodeInfos, nodesFromNotAutoscaledGroups, resourceLimiter)
 
   glog.V(0).Infof("kce  autoscale up resource left : %v",scaleUpResourcesLeft)
   if errLimits != nil {
      return nil, errLimits.AddPrefix("Could not compute total resources: ")
   }
 
   //Node在NodeGroup中但是没有Registed在kuber集群
   //计算方法  CurrentTarget - (readiness.Ready + readiness.Unready + readiness.LongNotStarted + readiness.LongUnregistered)
   upcomingNodes := make([]*schedulercache.NodeInfo, 0)
   for nodeGroup, numberOfNodes := range clusterStateRegistry.GetUpcomingNodes() {
      nodeTemplate, found := nodeInfos[nodeGroup]
      if !found {
         return nil, errors.NewAutoscalerError(
            errors.InternalError,
            "failed to find template node for node group %s",
            nodeGroup)
      }
 
      for i := 0; i < numberOfNodes; i++ {
         upcomingNodes = append(upcomingNodes, nodeTemplate)
      }
   }
 
   for upcomingnode := range upcomingNodes{
      glog.V(0).Infof("kce upcoming nodes :", upcomingnode)
   }
 
   podsPassingPredicates := make(map[string][]*apiv1.Pod)
   expansionOptions := make([]expander.Option, 0)
 
   //空 processor
   if processors != nil && processors.NodeGroupListProcessor != nil {
      var errProc error
      nodeGroups, nodeInfos, errProc = processors.NodeGroupListProcessor.Process(context, nodeGroups, nodeInfos, unschedulablePods)
      if errProc != nil {
         return nil, errors.ToAutoscalerError(errors.InternalError, errProc)
      }
   }
 
   for _, nodeGroup := range nodeGroups {
      //todo
      glog.V(0).Info("kce begin foreach nodeGroups : %s",nodeGroup.Id())
      // Autoprovisioned node groups without nodes are created later so skip check for them.
      if nodeGroup.Exist() && !clusterStateRegistry.IsNodeGroupSafeToScaleUp(nodeGroup.Id(), now) {
         glog.Warningf("Node group %s is not ready for scaleup", nodeGroup.Id())
         continue
      }
 
      currentTargetSize, err := nodeGroup.TargetSize()
      if err != nil {
         glog.Errorf("Failed to get node group size: %v", err)
         continue
      }
      if currentTargetSize >= nodeGroup.MaxSize() {
         // skip this node group.
         glog.V(4).Infof("Skipping node group %s - max size reached", nodeGroup.Id())
         continue
      }
 
      //get node template by node group
      nodeInfo, found := nodeInfos[nodeGroup.Id()]
 
      if !found {
         glog.Errorf("No node info (kce template info) for: %s", nodeGroup.Id())
         continue
      }
      //获取node group的recourse map
      glog.V(0).Infof("kce begin compute scale up resources delta, nodeGroup : %s ,nodeInfo : %v, resourceLimiter : %v", nodeGroup.Id(),nodeInfo,resourceLimiter)
      scaleUpResourcesDelta, err := computeScaleUpResourcesDelta(nodeInfo, nodeGroup, resourceLimiter)
      if err != nil {
         glog.Errorf("Skipping node group %s; error getting node group resources: %v", nodeGroup.Id(), err)
         continue
      }
 
      for k,v := range scaleUpResourcesDelta{
         glog.V(0).Infof("kce complate compute scale up resources delta key : %s ,value : %v ",k,v)
      }
 
      checkResult := scaleUpResourcesLeft.checkScaleUpDeltaWithinLimits(scaleUpResourcesDelta)
      if checkResult.exceeded {
         glog.V(4).Infof("Skipping node group %s; maximal limit exceeded for %v", nodeGroup.Id(), checkResult.exceededResources)
         continue
      }
 
      glog.V(0).Infof("kce scaleUpResourcesLeft node group %s; maximal limit exceeded for %v", nodeGroup.Id(), checkResult.exceededResources)
      option := expander.Option{
         NodeGroup: nodeGroup,
         Pods:      make([]*apiv1.Pod, 0),
      }
 
      //pod调度测试
      glog.V(0).Info("kce unscheduler pod begin scheduler test... nodeGroup : %s ,nodeInfo : %v",nodeGroup.Id(),nodeInfo)
      //返回可以调度的Pod
      option.Pods = FilterSchedulablePodsForNode(context, unschedulablePods, nodeGroup.Id(), nodeInfo)
 
      for _, pod := range option.Pods {
         glogx.V(1).UpTo(loggingQuota).Infof("Option Pod %s/%s will be scheduler ", pod.Namespace, pod.Name)
         podsRemainUnschedulable[pod] = false
      }
      passingPods := make([]*apiv1.Pod, len(option.Pods))
      copy(passingPods, option.Pods)
      podsPassingPredicates[nodeGroup.Id()] = passingPods
 
      if len(option.Pods) > 0 {
         //计算需要node
         if context.EstimatorName == estimator.BinpackingEstimatorName {
            binpackingEstimator := estimator.NewBinpackingNodeEstimator(context.PredicateChecker)
            option.NodeCount = binpackingEstimator.Estimate(option.Pods, nodeInfo, upcomingNodes)
            glog.V(0).Info("kce pod base estimator need node count : ",option.NodeCount)
         } else if context.EstimatorName == estimator.BasicEstimatorName {
            basicEstimator := estimator.NewBasicNodeEstimator()
            for _, pod := range option.Pods {
               basicEstimator.Add(pod)
            }
            option.NodeCount, option.Debug = basicEstimator.Estimate(nodeInfo.Node(), upcomingNodes)
            glog.V(0).Info("kce pod binpacking estimator : ",option.Debug)
            glog.V(0).Info("kce pod binpacking estimator need node count : ",option.NodeCount)
         } else {
            glog.Fatalf("Unrecognized estimator: %s", context.EstimatorName)
         }
         // append option node new count
         if option.NodeCount > 0 {
            expansionOptions = append(expansionOptions, option)
         } else {
            glog.V(0).Infof("No need for any nodes in %s", nodeGroup.Id())
         }
      } else {
         glog.V(0).Infof("No pod can fit to %s", nodeGroup.Id())
      }
   }
 
   //全部nodegroup，以及options
   if len(expansionOptions) == 0 {
      glog.V(1).Info("No expansion options")
      return &status.ScaleUpStatus{ScaledUp: false, PodsRemainUnschedulable: getRemainingPods(podsRemainUnschedulable)}, nil
   }
 
   // Pick some expansion option.
   // 根据expansion (random ,mostpods, price,waste)配置获取最佳伸缩组
   bestOption := context.ExpanderStrategy.BestOption(expansionOptions, nodeInfos)
   if bestOption != nil && bestOption.NodeCount > 0 {
      glog.V(0).Infof("Best option to resize: %s", bestOption.NodeGroup.Id())
      if len(bestOption.Debug) > 0 {
         glog.V(0).Info(bestOption.Debug)
      }
      glog.V(0).Infof("Estimated %d nodes needed in %s", bestOption.NodeCount, bestOption.NodeGroup.Id())
      //获取需要伸缩的newnode
      newNodes := bestOption.NodeCount
 
      if context.MaxNodesTotal > 0 && len(nodes)+newNodes > context.MaxNodesTotal {
         glog.V(0).Infof("Capping size to max cluster total size (%d)", context.MaxNodesTotal)
         newNodes = context.MaxNodesTotal - len(nodes)
         if newNodes < 1 {
            return nil, errors.NewAutoscalerError(
               errors.TransientError,
               "max node total count already reached")
         }
      }
 
      if !bestOption.NodeGroup.Exist() {
         oldId := bestOption.NodeGroup.Id()
         bestOption.NodeGroup, err = processors.NodeGroupManager.CreateNodeGroup(context, bestOption.NodeGroup)
         if err != nil {
            return nil, err
         }
         // Node group id may change when we create node group and we need to update
         // our data structures.
         if oldId != bestOption.NodeGroup.Id() {
            podsPassingPredicates[bestOption.NodeGroup.Id()] = podsPassingPredicates[oldId]
            delete(podsPassingPredicates, oldId)
            nodeInfos[bestOption.NodeGroup.Id()] = nodeInfos[oldId]
            delete(nodeInfos, oldId)
         }
      }
 
      nodeInfo, found := nodeInfos[bestOption.NodeGroup.Id()]
      if !found {
         // This should never happen, as we already should have retrieved
         // nodeInfo for any considered nodegroup.
         glog.Errorf("No node info for: %s", bestOption.NodeGroup.Id())
         return nil, errors.NewAutoscalerError(
            errors.CloudProviderError,
            "No node info for best expansion option!")
      }
 
      // apply upper limits for CPU and memory
      newNodes, err = applyScaleUpResourcesLimits(newNodes, scaleUpResourcesLeft, nodeInfo, bestOption.NodeGroup, resourceLimiter)
      if err != nil {
         return nil, err
      }
      // targetNodeGroup 即为bestOption的NodeGroup
      targetNodeGroups := []cloudprovider.NodeGroup{bestOption.NodeGroup}
      if context.BalanceSimilarNodeGroups {
         similarNodeGroups, typedErr := nodegroupset.FindSimilarNodeGroups(bestOption.NodeGroup, context.CloudProvider, nodeInfos)
         if typedErr != nil {
            return nil, typedErr.AddPrefix("Failed to find matching node groups: ")
         }
         similarNodeGroups = filterNodeGroupsByPods(similarNodeGroups, bestOption.Pods, podsPassingPredicates)
         glog.V(0).Infof("kce fine node similar group %v ", similarNodeGroups)
         for _, ng := range similarNodeGroups {
            if clusterStateRegistry.IsNodeGroupSafeToScaleUp(ng.Id(), now) {
               targetNodeGroups = append(targetNodeGroups, ng)
            } else {
               // This should never happen, as we will filter out the node group earlier on
               // because of missing entry in podsPassingPredicates, but double checking doesn't
               // really cost us anything
               glog.V(2).Infof("Ignoring node group %s when balancing: group is not ready for scaleup", ng.Id())
            }
         }
         if len(targetNodeGroups) > 1 {
            var buffer bytes.Buffer
            for i, ng := range targetNodeGroups {
               if i > 0 {
                  buffer.WriteString(", ")
               }
               buffer.WriteString(ng.Id())
            }
            glog.V(0).Infof("Splitting scale-up between %v similar node groups: {%v}", len(targetNodeGroups), buffer.String())
         }
      }
      scaleUpInfos, typedErr := nodegroupset.BalanceScaleUpBetweenGroups(
         targetNodeGroups, newNodes)
      if typedErr != nil {
         return nil, typedErr
      }
      glog.V(0).Infof("Final scale-up plan: %v", scaleUpInfos)
      for _, info := range scaleUpInfos {
         typedErr := executeScaleUp(context, clusterStateRegistry, info, gpu.GetGpuTypeForMetrics(nodeInfo.Node(), nil))
         if typedErr != nil {
            return nil, typedErr
         }
      }
 
      clusterStateRegistry.Recalculate()
      glog.V(0).Info("kce cloud provider end scale up ***********************")
      return &status.ScaleUpStatus{
            ScaledUp:                true,
            ScaleUpInfos:            scaleUpInfos,
            PodsRemainUnschedulable: getRemainingPods(podsRemainUnschedulable),
            PodsTriggeredScaleUp:    bestOption.Pods,
            PodsAwaitEvaluation:     getPodsAwaitingEvaluation(unschedulablePods, podsRemainUnschedulable, bestOption.Pods)},
         nil
   }
 
   return &status.ScaleUpStatus{ScaledUp: false, PodsRemainUnschedulable: getRemainingPods(podsRemainUnschedulable)}, nil
}
```