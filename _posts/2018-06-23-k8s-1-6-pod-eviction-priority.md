---
title: k8s 1.6 pod eviction priority
description: 
categories:
- k8s
tags:
- policy
---

[之前文章](https://larryck.github.io/k8s/2018/06/16/K8S-1-6下提高关键pod优先级的方法/)提到的使用critical pod的方式保障关键pod的方式因为需要将pod放到kube-system namespace会带来诸多不便。

为了方便关键应用尽量使用系统资源并且在eviction时能够尽可能保证其不被kill掉，我这边增加了eviction priority功能。

Eviction-priority通过pod的ck/eviction-priority annotation开启，ck/eviction-priority设置值介于-999与999之间，值越小保留的优先级越高。没有设置或者设置错误的默认优先级是0。当需要eviction时，相同的service quality优先踢除优先级低的pod。
      
建议平台的用户应用使用Burstable service quality，并且配置相应的eviction priority。