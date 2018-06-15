---
title: heapster metrics的一些疑惑
description: 
categories:
- k8s
tags:
- metrics
---

本文解决以下问题：
>- heapster node metrics如何获得？
>- heapster CPU_USAGE与top中的各个利用率之间的关系是？


Heapster使用kubelet client从/stats/container/接口请求cadvisor的container性能数据，如果container name为'/'的话则判断为是node数据。

node性能数据的获取方式与其他container的性能数据一致，cpu及内存数据从cgroupfs中获得，网络数据从/proc/pid/net中获得。可以参见文章[cadvisor数据采集]()。

cpu_usage在cgroup中的值是当前namespace的cpu使用时间，包括用户态使用时间与内核态使用时间。cgroup的时间统计了cgroup中所有进程的CPU使用信息，top命令则是获取了/proc/中所有pid的cpu使用信息。这两个统计信息是一致的。
我们可以认为cpu_usage是top命令的所有用户态及内核态CPU使用值之和。

