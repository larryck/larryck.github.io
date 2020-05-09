---
title: cinder csi block number issue
description: 
categories:
- k8s
- storage
tags:
- policy
---

# Problem
The problem began with a pod stucking at creating stage for a long time. After studying it, we found that PV used by the pod fails to attach to Kubernetes node, which is abnormal because the cinder csi system has performed well for a long time.

# Analysis
We tried to find out what happened. 

First we checked the log of the provisioner and attacher of our cinder csi driver. The log showed that provisioning cloud block was ok but trying to bind the block to target VM server ended up with error. We checked the log of openstack further. Below is what we find from nova log:
```
NovaException: No free disk device name for prefix 'vd'
Unexpected exception in API method
```
From openstack dashboard we knew that the VM server has already attached 25 cloud block devices and device descriptors from vda-vdz has already used up.

We can also confirm this issue from openstack community. A bug titled "**add volume fails over 26vols and returns 500 API error with libvirt driver**" has already been submited to nova community.

# Solution
The problem reveals some restriction of the IaaS platform we adopted, and we are working with them to solve it.

In the meantime, the problem pushes us to provide filesystem based csi solution for largescale volume usage scenarios, which is the lks-nfs-csi solution. We'll talk about it in the coming blogs. 