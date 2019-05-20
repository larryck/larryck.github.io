---
title:  runc disable kmem accounting
description: 
categories:
- k8s
tags:
- bug
---

在[kernel deadlock caused by kmem limit in k8s](https://larryck.github.io/k8s/2019/05/10/kernel-deadlock-caused-by-kmem-limit-in-k8s/)一文中我们提到了可以通过升级至Disable kmem accounting特性的18.09.1版本以上docker的方法解决问题。然而，当我们升级docker之后，通过memory.kmem.slabinfo发现kernel memory accounting依旧开启着。这篇文章将分析背后的原因。

# runc
在runc中，libcontainer根据cgroup的驱动不同分为cgroupfs以及systemd。

我们先看18.09.1 docker对应的runc-v1.0.0-rc6中libcontainer systemd对kmem accounting的实现。我们可以看到以下代码：
```
	// We have to set kernel memory here, as we can't change it once
	// processes have been attached to the cgroup.
	if c.Resources.KernelMemory != 0 {
		if err := setKernelMemory(c); err != nil {
			return err
		}
	}
```
依旧是说，除非已经设置了KernelMemory，默认情况下是不会启用kmem accounting机制的，这与docker的release描述是一致的。

然而，我们细查发现，当前问题系统中我们默认的cgroup driver用的是cgroupfs。对于cgroupfs，libcontainer的实现分为enable和disable kmem两种情况，两者通过// +build linux,!nokmem和// +build linux,nokmem进行区分。也就是说，在使用cgroupfs时，如果我们想要disable kmem，需要在编译runc时使用nokmem编译选项。

显然，我们使用的runc版本并没有使用这个选项，从而造成了我们现在升级了docker后kmem accounting机制依旧enable的问题。
