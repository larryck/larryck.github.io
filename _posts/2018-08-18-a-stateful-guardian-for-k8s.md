---
title: a stateful guardian for k8s
description: 
categories:
- k8s
tags:
- statefulset
---

Many people believe that k8s designs for stateless apps. When we talk about those exciting features, such as replicas and HPA, mostly we mean stateless apps. Though there are still lots of arguments on whether k8s suits for stateful apps, we are trying to build our computing platform, which supports stateful apps of course, base on it. We create a StatefulGuardian resource type to manage each statefulset object, which guards the stability and the viability of each statefulset pods. In this article, I will describe how we extend the k8s statefulset object. 

We think that stateful object provides some basic features need to construct a stateful running environment, for example, ordered start/shutdown process, fixed hostname, etc. But there are several aspects that we can improve, I’ll describe below. We have developed the StatefulGuardian project which implements some basic features we proposal, it’s open sourced [here](https://github.com/larryck/StatefulGuardian), and contributions to the project are welcomed.

# lifecycle management
Stateful apps are complicated, more specific application knowledge are needed to manage their whole lifecycle. Suppose we are trying to build a Mysql statefulset cluster with several pods, we first need to make sure that all these pods can join in one cluster, but that’s only the first step, other work include: how do one failed pod rejoin in the cluster, how to shut down the whole cluster and start it normally again later, how to gracefully update the cluster, etc. 

K8s provides some basic features to fulfill basic platform level actions, for example, stateful controller will restart the failed pod automatically. But it leaves all other restart logics to application builders which makes the application containerization process complicated. In StatefulGuardian, we expose standard lifecycle handlers for application developers to conduct the application lifecycle in their own language, for example cluster initial/pod restart/cluster restart/cluster shutdown/runtime tasks injection etc. Application maintainers do not need to know much about container or k8s, they write scripts/commands for each scenery they need and StatefulGuardian runs these scripts under the right occasions.

# application level monitoring
We need more monitoring perspective to debug performance issues of stateful apps. It will be cool if there is a way that we can get performance metrics at runtime and do not disturb the application in the meantime. 

In StatefulGuardian, we integrate StatefulGuardian, Prometheus and dashboard to build a metrics channel for statefulset applications. Maintainers can add metrics extraction commands to StatefulGuardian by simply updating the StatefulGuardian object, then it will get those metrics, push them to Prometheus and you can see those metrics from dashboard.

# application viability
Statefulset application failure has a large impact, it is always related to data reload, data rebalance or date loss, which will cause either performance issue or business disaster. 

In StatefulGuardian, we try our best to keep stateful applications correct and alive. First, we manage priorities for stateful apps. The priority of stateful is much higher than stateless apps, which make sure that stateful pods can be remained through resource exhausting pod eviction. Second, we manage performance for stateful applications. StatefulGuardian watches resource usages of each stateful application and take actions to prevent it from resource exhaustion, such as traffic control. 

Other advanced features such as application freezing, runtime volume snapshot and failure track are under discussion.


