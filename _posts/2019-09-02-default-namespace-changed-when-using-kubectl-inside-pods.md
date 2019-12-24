---
title: default namespace changed when using kubectl inside pods
description: 
categories:
- k8s
tags:
- policy
---

# 问题
当我们在某个pod中使用kubectl访问Kubernetes集群时，kubectl访问的default namespace将指定为该pod所在的namespace。

# 分析
在Kubernetes集群内部执行的应用可以通过serviceaccount访问apiserver。serviceaccount绑定在pod的/var/run/secrets/kubernetes.io/serviceaccount目录下，包括ca.crt,namespace , token三个文件，其中ca.crt/token两个文件包含认证及RBAC权限验证信息，namespace文件即是当前pod所执行的namespace。

当我们使用在pod内部使用kubectl时，kubectl默认使用serviceaccount认证，因而即使不传递kubeconfig文件也可以访问k8s的资源。

但是这里存在的一个问题是，即使用户指定kubeconfig文件访问其他集群时，kubectl也将使用serviceaccount的namespace作为访问的default namespace，这可能是一个Bug。

更多关于serviceaccount的信息可以查看[这篇文章](https://medium.com/better-programming/k8s-tips-using-a-serviceaccount-801c433d0023)。