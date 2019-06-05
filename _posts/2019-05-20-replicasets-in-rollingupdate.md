---
title:  old replicasets not deleted when rolling update deployments
description: 
categories:
- k8s
tags:
- bug
---
Recently I found that after rolling update deployments for lots of times, thousands of old replicasets remained in my k8s cluster. Why are old replicasets not deleted in k8s?

# Revision History Limit
A Deployment’s revision history is stored in the ReplicaSets it controls.

.spec.revisionHistoryLimit is an optional field that specifies the number of old ReplicaSets to retain to allow rollback. These old ReplicaSets consume resources in etcd and crowd the output of kubectl get rs. The configuration of each Deployment revision is stored in its ReplicaSets; therefore, once an old ReplicaSet is deleted, you lose the ability to rollback to that revision of Deployment. By default, 10 old ReplicaSets will be kept, however its ideal value depends on the frequency and stability of new Deployments.

But in my cluser, there were hundreds of replicasets for each deployment, which was of cause not the case of default of 10. Why did that happen?

#  revisionHistoryLimit in deployment extensions/v1beta1
After searching in k8s source code I found that the default value of revisionHistoryLimit is different from later resource versions. 

For version extensions/v1beta1, DeploymentSpec has RevisionHistoryLimit implemented below:

```
	// The number of old ReplicaSets to retain to allow rollback.
	// This is a pointer to distinguish between explicit zero and not specified.
	// This is set to the max value of int32 (i.e. 2147483647) by default, which
	// means "retaining all old RelicaSets".
	// +optional
	RevisionHistoryLimit *int32 `json:"revisionHistoryLimit,omitempty" protobuf:"varint,6,opt,name=revisionHistoryLimit"
```

We can see that k8s keeps all old replicasets, which explains what happened in my case. But in other versions, the default value of RevisionHistoryLimit totally depends. For example, in version apps/v1beta1 it is 2, but in app/v1 it's 10.