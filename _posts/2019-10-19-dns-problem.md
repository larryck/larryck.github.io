---
title: k8s dns problem
description: 
categories:
- k8s
- dns
tags:
- policy
---

# 问题
在我们的集群中，每个机器有内网、外网两个网卡。其中，外网有间断性的断网问题，因而切换了k8s、kubelet、flannel使用内部网络。然而，应用依旧检测到间断性断网。

经过各种复杂的问题分析，最后定位到是dns问题。下面是一些日志：

```
  [ERROR] plugin/errors: 2 articleservice.blogservice.svc.cluster.local.labs.com. A: unreachable backend: read udp 10.244.0.217:34688->10.240.0.10:     53: i/o timeout
  [ERROR] plugin/errors: 2 translateservice.blogservice.svc.cluster.local.labs.com. A: unreachable backend: read udp 10.244.0.217:45144->10.240.0.11: 53: i/o timeout
```

物理机的的reslov.conf配置如下：

```
# Generated by NetworkManager
search labs.com
nameserver 10.240.0.10
nameserver 10.240.0.11
```

这里存在的问题是：**为什么articleservice.blogservice服务域名需要到物理机的nameserver解析？为什么coredns未解析service dns？**

# 一些信息

这篇文章[Kubernetes and Azure AKS 1.11+: coredns external random DNS resolution errors after cluster upgrade](https://www.sesispla.net/en/azure-aks-1-11-random-dns-resolution-error-after-cluster-upgrade/)提到了coredns在 

```
under high load scenarios, coredns is using on all the DNS servers defined at VNET level 
```

这个问题需要进一步研究。