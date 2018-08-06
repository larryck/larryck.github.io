---
title: pod graceful deletion流程中的 terminating
description: 
categories:
- k8s
tags:
- policy
---

Terminating步骤是kubectl在删除pod时常见的一个步骤，但是这个状态并不是k8s pods对象自身的一个状态。这篇文章回答两个问题：
>- Terminating步骤代表什么？
>- 为什么有的pod删除几乎看不到Terminating步骤，有的pod长时间处于terminating步骤？

# Terminating

首先，从kubectl的这段代码（printers\internalversion\printers.go）我们可以看到，出现terminating的原因是pod的DeletionTimestamp属性不为空。

	// AddHandlers adds print handlers for default Kubernetes types dealing with internal versions.
	func AddHandlers(h *printers.HumanReadablePrinter) {
		h.Handler(podColumns, podWideColumns, printPodList)
		h.Handler(podColumns, podWideColumns, printPod)
		h.Handler(podTemplateColumns, nil, printPodTemplate)
		h.Handler(podTemplateColumns, nil, printPodTemplateList)


	func printPod(pod *api.Pod, w io.Writer, options printers.PrintOptions) error {
		if err := printPodBase(pod, w, options); err != nil {
			return err
		}
	
		return nil
	}

	func printPodBase(pod *api.Pod, w io.Writer, options printers.PrintOptions) error {
	    .......
		} else if pod.DeletionTimestamp != nil {
			reason = "Terminating"
		}

DeletionTimestamp属性（api\types.go）的解释是Pod删除的时间戳，这个时间戳主要为pod的graceful termination属性设计（如果未没有设置，k8s默认为30s）。

	// DeletionTimestamp is RFC 3339 date and time at which this resource will be deleted. This
	// field is set by the server when a graceful deletion is requested by the user, and is not
	// directly settable by a client. The resource is expected to be deleted (no longer visible
	// from resource lists, and not reachable by name) after the time in this field. Once set,
	// this value may not be unset or be set further into the future, although it may be shortened
	// or the resource may be deleted prior to this time. For example, a user may request that
	// a pod is deleted in 30 seconds. The Kubelet will react by sending a graceful termination
	// signal to the containers in the pod. After that 30 seconds, the Kubelet will send a hard
	// termination signal (SIGKILL) to the container and after cleanup, remove the pod from the
	// API. In the presence of network partitions, this object may still exist after this
	// timestamp, until an administrator or automated process can determine the resource is
	// fully terminated.
	// If not set, graceful deletion of the object has not been requested.
	//
	// Populated by the system when a graceful deletion is requested.
	// Read-only.
	// More info: http://releases.k8s.io/HEAD/docs/devel/api-conventions.md#metadata
	// +optional
	DeletionTimestamp *metav1.Time

DeletionTimestamp时间戳在pod初次delete时设置（apiserver\pkg\registry\rest\delete.go），

	// BeforeDelete tests whether the object can be gracefully deleted. If graceful is set the object
	// should be gracefully deleted, if gracefulPending is set the object has already been gracefully deleted
	// (and the provided grace period is longer than the time to deletion), and an error is returned if the
	// condition cannot be checked or the gracePeriodSeconds is invalid. The options argument may be updated with
	// default values if graceful is true. Second place where we set deletionTimestamp is pkg/registry/generic/registry/store.go
	// this function is responsible for setting deletionTimestamp during gracefulDeletion, other one for cascading deletions.
	func BeforeDelete(strategy RESTDeleteStrategy, ctx genericapirequest.Context, obj runtime.Object, options *metav1.DeleteOptions) (graceful, gracefulPending bool, err error) {
            ......
            now := metav1.NewTime(metav1.Now().Add(time.Second * time.Duration(*options.GracePeriodSeconds)))
			objectMeta.SetDeletionTimestamp(&now)
			objectMeta.SetDeletionGracePeriodSeconds(options.GracePeriodSeconds)


apiserver\pkg\registry\generic\registry\store.go

	// Delete removes the item from storage.
	func (e *Store) Delete(ctx genericapirequest.Context, name string, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
	    ......
		graceful, pendingGraceful, err := rest.BeforeDelete(e.DeleteStrategy, ctx, obj, options)

# terminating 时间
另一个问题是为什么有的pod立马就删除了而有的pod在terminating状态等待一段时间？

我们知道，k8s对pod的删除包括prestop handler的执行、pod中container的删除等。而docker对container的删除分为两步，分别由两个信号量SIGTERM及SIGKILL处理。在graceful termination流程中，k8s首先向pod中的container发送SIGTERM信号，等待gracefulPeriodSeconds之后再发送SIGKILL信号。

不考虑prestop handler的执行，区别出现在container对SIGTERM的处理上，如果Container中的1号进程忽略了SIGTERM的处理或者在收到SIGTERM时未退出，则Container将一直等待grancePeroid之后由SIGKILL结束，这将造成Pod一直处于Terminating状态一段时间。

# graceful delete的时间组成

在pod的graceful deletion流程中涉及到以下几部分时间：

- prestop: prestop是pod在结束之前执行的handler，如果prestop的执行时间达到gracePeriod（系统默认为30s）时prestop程序的执行将被终止。 prestop的执行时间算作gracePeriod的一部分。
- minimumGracePeriodInSeconds：如果prestop的执行耗尽了gracePeriod使得剩余时间小于minimumGracePeriodInSeconds，gracePeriod将再延时至minimumGracePeriodInSeconds。minimumGracePeriodInSeconds默认为2s。
- docker StopContainer：k8s将剩余的时间作为docker的graceful period调用docker的stop container流程。

从上面流程我们可以看出，k8s的pod delete时间默认最长能够达到32秒。

