---
title: how statefulset pod deletion is different from the common graceful deletion
description: 
categories:
- k8s
tags:
- statefulset
---

After a long-time delay, now I'll keep on investigating some confusion of kubernetes' key components. Today’s topic is: **Is there any difference between the deletion process of statefulset pods and pods of other kinds?** 

As we know that statefulset applications are always crucial ones, the kubernetes cluster may want to pay more attention to serve these apps well, especially the deletion process which can easily cause data loss if we do not deal with it gracefully. 

# Graceful deletion#
The kubernetes provides graceful deletion which gives apps some time to clear environment before being force-killed. The graceful-period defaults to be 30 seconds, but users can change it. After graceful-period, kubernetes will kill the pod, destroy all containers and do not check whether the clearing process finishes or not. 

The graceful deletion process, for me, is kind of too simple. We can not precisely know how much time the app needs to shut down. It varies from different loads the system has and different requests the app serves. 
I think kubernetes may have a more advanced deletion policy related to stateful apps, so I’ll try to find out that piece of code. 

# Statefulset controller#
Statefulset controller works inside the controller manager which manages all stateful apps in the cluster. The deletion policy, **DeleteStatefulPod**, is part of the **UpdateStatefulSet** function, it takes over control when it detects failed pods or there are more pods than the statefulset desires. 

The **realStatefulPodControl** structure implements DeleteStatefulPod policy. Below is the code :

```

	func (spc *realStatefulPodControl) DeleteStatefulPod(set *apps.StatefulSet, pod *v1.Pod) error {
	    err := spc.client.Core().Pods(set.Namespace).Delete(pod.Name, nil)
	    spc.recordPodEvent("delete", set, pod, err)
	    return err
	}

```	

As we can see from the code, **there is nothing different here**, it just calls the normal pod deletion API. 

Currently, we can set a large-enough graceful period to give stateful apps more time to gracefully shutdown themselves, but it will be great if we build more targeted policy. For example, some apps may fail during the clearing process, and we do not want it to be force-deleted when that happens. More work needs to do to form a more reasonable solution.
