---
title: port forwarding reported tcp dial failure in LKS
description: 
categories:
- k8s
- network
tags:
- policy
---

# 问题
在LKS托管集群中，用户在使用kubectl port-forward时遇到 error upgrading connection: error dialing backend: dial tcp xxxxx:10250 timeout问题。

# 分析
使用port-forward功能时，kubectl首先与apiserver通信，通过apiserver将信息通过kubelet 10250端口转发到目标pod。

LKS出现这个问题原因在于为了保证集群安全，LKS在部署过程中将Kube0部署在admin tenant的独立子网中，这将导致在用户Kubernetes集群的apiserver与kubelet位于不同的网段。Kubelet通过public IP与apiserver交互不会存在问题，但是反过来apiserver通过内网地址访问Kubelet时网络连接是无法到达的。