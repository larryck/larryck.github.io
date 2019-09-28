---
title:  how replicasets choose pods to delete when scale down
description: 
categories:
- k8s
tags:
- policy
---

# 问题：
当replicasets需要删除某些pod时，controller将如何选择哪些pod？

# Pod的选择策略
当当前pod数量大于replicaset spec中期望的数量时，replicasset controller开始选择需要删除的pod。

*pkg/controller/replicaset/replica_set.go*
```
		// Choose which Pods to delete, preferring those in earlier phases of startup.
		podsToDelete := getPodsToDelete(filteredPods, diff)


func getPodsToDelete(filteredPods []*v1.Pod, diff int) []*v1.Pod {
	// No need to sort pods if we are about to delete all of them.
	// diff will always be <= len(filteredPods), so not need to handle > case.
	if diff < len(filteredPods) {
		// Sort the pods in the order such that not-ready < ready, unscheduled
		// < scheduled, and pending < running. This ensures that we delete pods
		// in the earlier stages whenever possible.
		sort.Sort(controller.ActivePods(filteredPods))
	}
	return filteredPods[:diff]
}
```

controller使用一个sort来排序所有该replicaseta管理的pod，并从后面开始删除需要处理的pod。

接下来，我们主要关注controller对pod的排序策略。

*pkg/controller/controller_utils.go*
```
// ByLogging allows custom sorting of pods so the best one can be picked for getting its logs.
type ByLogging []*v1.Pod


func (s ByLogging) Less(i, j int) bool {
	// 1. assigned < unassigned
	if s[i].Spec.NodeName != s[j].Spec.NodeName && (len(s[i].Spec.NodeName) == 0 || len(s[j].Spec.NodeName) == 0) {
		return len(s[i].Spec.NodeName) > 0
	}
	// 2. PodRunning < PodUnknown < PodPending
	m := map[v1.PodPhase]int{v1.PodRunning: 0, v1.PodUnknown: 1, v1.PodPending: 2}
	if m[s[i].Status.Phase] != m[s[j].Status.Phase] {
		return m[s[i].Status.Phase] < m[s[j].Status.Phase]
	}
	// 3. ready < not ready
	if podutil.IsPodReady(s[i]) != podutil.IsPodReady(s[j]) {
		return podutil.IsPodReady(s[i])
	}
	// TODO: take availability into account when we push minReadySeconds information from deployment into pods,
	//       see https://github.com/kubernetes/kubernetes/issues/22065
	// 4. Been ready for more time < less time < empty time
	if podutil.IsPodReady(s[i]) && podutil.IsPodReady(s[j]) && !podReadyTime(s[i]).Equal(podReadyTime(s[j])) {
		return afterOrZero(podReadyTime(s[j]), podReadyTime(s[i]))
	}
	// 5. Pods with containers with higher restart counts < lower restart counts
	if maxContainerRestarts(s[i]) != maxContainerRestarts(s[j]) {
		return maxContainerRestarts(s[i]) > maxContainerRestarts(s[j])
	}
	// 6. older pods < newer pods < empty timestamp pods
	if !s[i].CreationTimestamp.Equal(&s[j].CreationTimestamp) {
		return afterOrZero(&s[j].CreationTimestamp, &s[i].CreationTimestamp)
	}
	return false
}
```
从上面的代码我们可以看到，controller的删除顺序是未调度、pending、Unknown、Ready、Ready时间少、重启次数多、创建时间短的Pod。

# 讨论
这个删除策略当然在考虑通用情况时是OK的。然而，这也暴露了更多应用风险。因为对于应用来说保证长时间稳定运行是更加困难的，而我们每次删除时都选择最新的Pod，这隐含着对应用的长时间稳定运行显然有更高的要求。

社区中应该考虑对这种情形设置一些删除策略。
