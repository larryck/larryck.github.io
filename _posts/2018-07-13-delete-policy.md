---
title: k8s delete policy
description: 
categories:
- k8s
tags:
- statefulset
---

In k8s 1.12, the default policy for objects will be orphan. For example, the code below delete statefulset **ss** but leaves all pods belongs to it:



	client.AppsV1beta1().StatefulSets(ss.Namespace).Delete(ss.Name, &metav1.DeleteOptions{})


We have to set **PropagationPolicy** explicitly to delete pods along with the statefulset.

	backgroundPolicy := metav1.DeletePropagationBackground
    client.AppsV1beta1().StatefulSets(ss.Namespace).Delete(ss.Name, &metav1.DeleteOptions{PropagationPolicy: &backgroundPolicy})
