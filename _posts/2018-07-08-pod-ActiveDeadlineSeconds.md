---
title: pod activeDeadlineSeconds
description: 
categories:
- k8s
tags:
- policy
---

在分析另一个问题时误打误撞看一下activeDeadlineSeconds的代码。这里做个记录。

# ActiveDeadlineSeconds

这个属性设置Pod从开始执行以来的最大Active秒数（api\types.go）：

	// Optional duration in seconds relative to the StartTime that the pod may be active on a node
	// before the system actively tries to terminate the pod; value must be positive integer
	// +optional
	ActiveDeadlineSeconds *int64

kubelet判断是否需要evict pod时，pod执行时间是否超过activeDeadlineSeconds也是一个衡量标准（kubelet\active_deadline.go）：

	// ShouldEvict returns true if the pod is past its active deadline.
	// It dispatches an event that the pod should be evicted if it is past its deadline.
	func (m *activeDeadlineHandler) ShouldEvict(pod *v1.Pod) lifecycle.ShouldEvictResponse {
		if !m.pastActiveDeadline(pod) {
			return lifecycle.ShouldEvictResponse{Evict: false}
		}
		m.recorder.Eventf(pod, v1.EventTypeNormal, reason, message)
		return lifecycle.ShouldEvictResponse{Evict: true, Reason: reason, Message: message}
	}

	// pastActiveDeadline returns true if the pod has been active for more than its ActiveDeadlineSeconds
	func (m *activeDeadlineHandler) pastActiveDeadline(pod *v1.Pod) bool {
		// no active deadline was specified
		if pod.Spec.ActiveDeadlineSeconds == nil {
			return false
		}
		// get the latest status to determine if it was started
		podStatus, ok := m.podStatusProvider.GetPodStatus(pod.UID)
		if !ok {
			podStatus = pod.Status
		}
		// we have no start time so just return
		if podStatus.StartTime.IsZero() {
			return false
		}
		// determine if the deadline was exceeded
		start := podStatus.StartTime.Time
		duration := m.clock.Since(start)
		allowedDuration := time.Duration(*pod.Spec.ActiveDeadlineSeconds) * time.Second
		return duration >= allowedDuration
	}





