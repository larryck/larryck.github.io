---
title: statefulset hostname造成的kafka集群异常
description: 
categories:
- k8s
tags:
- statefulset
---


在k8s环境中部署kafka过程中使用了基于Deployments及Statefuelsets两种方式部署。然而，发现Deployments集群状态正常，而statefulsets集群使用时则返回Leader未选举状态。

# 部署情况
创建了ka-1到ka-4四个服务，伴随四个在同一个namespace的kafka Deployments或Statefulset。Zookeeper后端使用3个zookeeper Deployments或Statefulset实例。

通过hostname属性为Deployments或statefulset指定pod的hostname。

# 错误分析
在Deployments环境中，通过netstat能看到ka-0有多个9092端口的连接，来自于ka-1/ka-2/ka-3。而Statefulset环境中看不到对应的9092连接。

通过查询我发现，kafka使用hostname作为集群中节点的标识。这是造成整个问题的原因。在Deployments的Pod中我们查询hostname得到的时ka-**x**，而Statefulset中的Pod对应的hostname为ka-**x**-0。 由于两个集群的service name都是ka-**x**，在statefulset集群中，kafka实例通过ka-**x**-0域名连接其他实例时网络是不同的。

因而，对于statefulset集群，service name需要修改成ka-**x**-0。

这意味着，虽然statefulset的spec podspec中是有hostname属性的，但是这个属性并不生效，statefulset依旧会用$(statefulset name)-$(ordinal)作为name。