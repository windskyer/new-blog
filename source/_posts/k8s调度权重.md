---
title: k8s调度权重
date: 2019-03-01 16:21:27
tags: scheduler
categories: Kubernetes
---

# k8s 调度算法

算法需要经过两个阶段，分别是过滤和打分，首先过滤掉一部分，保证剩余的节点都是可调度的，接着在打分阶段选出最高分节点，该节点就是scheduler的输出节点。

>1. 过滤规则
>2. 计算权重

**[过滤规则链接地址](http://www.flftuu.com/2019/02/28/algorithm/)**

<!-- more -->

## 计算权重

### 默认注册权重函数13个

|策略名称 |权重值Weight|备注|
|---|---|---|
|SelectorSpreadPriority|1|default|
|InterPodAffinityPriority|1|default|
|LeastRequestedPriority|1|default|
|BalancedResourceAllocation|1|default|
|NodePreferAvoidPodsPriority|10000|default|
|NodeAffinityPriority|1|default|
|TaintTolerationPriority|1|default|
|ImageLocalityPriority|1|default|
|ServiceSpreadingPriority|1|default|
|EqualPriority|1|default|
|MostRequestedPriority|1|default|
|RequestedToCapacityRatioPriority|1|default|
|ResourceLimitsPriority|1|开启ResourceLimitsPriorityFunction|

**ClusterAutoscalerProvider 调度器把LeastRequestedPriority 替换成MostRequestedPriority 策略**

### 打分源码

```go
// PriorityConfig is a config used for a priority function.
type PriorityConfig struct {
	Name   string
	Map    PriorityMapFunction
	Reduce PriorityReduceFunction
	// TODO: Remove it after migrating all functions to
	// Map-Reduce pattern.
	Function PriorityFunction
	Weight   int
}
```

>1. Name 打分函数名称
>2. Map 加分方法
>3. Reduce 减分方法
>4. Function 通用方法，将废弃
>5. Weight 打分函数的权重值

**打分调度顺序 Function > Map > Reduce**
**优先计算所有的通用性方法Fuction， 再计算打分函数的加分方法，最后计算所有打分函数的加分方法**

### SelectorSpreadPriority

>- 加分方法：CalculateSpreadPriorityMap
>- 减分方法: CalculateSpreadPriorityReduce

#### 加分方法：CalculateSpreadPriorityMap

>1. 获取正在调度pod 有关的svc，sts，rs，rc
>2. 获取该node 上的所有pod
>3. 存在的pod 与正在调度的pod 是同一svc或者sts，rs，rc 则改node Score分数 +1

#### 减分方法: CalculateSpreadPriorityReduce

>1. 获取加分方法计算出所有node 的分数
>2. 获取所有node 中的 最高分数
>3. 如果其中有node 设置region和zone， 则获取每一个"regino::zone" 其中最高的分数
>4. 获取分数常数值 MaxPriority = 10, zoneWeighting float64 = 2.0 / 3.0
>5. fScore = MaxPriorityFloat64 * (float64(maxCountByNodeName-result[i].Score) / maxCountByNodeNameFloat64) 注释：--> 常熟10 * (最大值 - 当前node的值) / 最大值
>6. score 值，通过上面计算方法则表示 在加分方法中分数越高的node 最后分数越低
>7. 如果其中有region和zone设置, 则计算公式 zoneScore = MaxPriorityFloat64 * (float64(maxCountByZone-countsByZone[zoneID]) / maxCountByZoneFloat64)， fScore = (fScore * (1.0 - zoneWeighting)) + (zoneWeighting * zoneScore) 注释--> (常熟10 * (zone最大值 - 当前zone的值) / zone最大值) * 2/3 + node的fscore * 1/3
>8. score值，通过上面计算法则表示 在加分方法中分数越高的node 最后分数越低 和 不同zone的node 分数越高

**总结： 相同service／rc的pods越分散，得分越高， 不通zone的node 分数越高**

### InterPodAffinityPriority

>- 通用方法 CalculateInterPodAffinityPriority

#### 通用方法 CalculateInterPodAffinityPriority

- 获取调度pod 上的PodAffinity 和PodAntiAffinity
  - 需调度pod上有 PodAffinity 和PodAntiAffinity设置
    - 获取该node上所有pod信息
    - 
- 

**总结： 通过迭代 weightedPodAffinityTerm 的元素计算和，并且如果对该节点满足相应的PodAffinityTerm，则将 “weight” 加到和中，具有最高和的节点是最优选的。**


### LeastRequestedPriority

>- 加分方法 leastResourcePriority.PriorityMap

#### 加分方法 leastResourcePriority.PriorityMap

### BalancedResourceAllocation

>- 加分方法 balancedResourcePriority.PriorityMap

#### 加分方法 balancedResourcePriority.PriorityMap

### NodePreferAvoidPodsPriority

> 加分方法 priorities.CalculateNodePreferAvoidPodsPriorityMap

#### 加分方法 priorities.CalculateNodePreferAvoidPodsPriorityMap

### NodeAffinityPriority

>- 加分方法 priorities.CalculateNodeAffinityPriorityMap
>- 减分方法 priorities.CalculateNodeAffinityPriorityReduce

### TaintTolerationPriority

>- 加分方法 priorities.ComputeTaintTolerationPriorityMap
>- 减分方法 priorities.ComputeTaintTolerationPriorityReduce

### ImageLocalityPriority

>- 加分方法 riorities.ImageLocalityPriorityMap

### ServiceSpreadingPriority

>- 通 SelectorSpreadPriority 相同，以被代替

### EqualPriority

>- 加分方法 core.EqualPriorityMap

### MostRequestedPriority

>- 加分方法 priorities.MostRequestedPriorityMap

### RequestedToCapacityRatioPriority

>- 加分方法 priorities.RequestedToCapacityRatioResourceAllocationPriorityDefault().PriorityMap

### ResourceLimitsPriority

>- 加分方法 priorities.ResourceLimitsPriorityMap