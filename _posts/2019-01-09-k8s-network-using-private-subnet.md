---
title: k8s network using private subnet
description: 
categories:
- k8s
- network
tags:
- policy
---

# 问题
在具有多网卡的K8s集群中，Kubelet和flannel默认会选择第一块网卡作为K8s集群的通信网络。当你需要指定K8s使用其他网卡时，应该怎么操作。

这其中涉及到对K8s、Kubelet以及Flannel的配置。

# K8s
K8s需要指定apiserver的访问接口，以及配置证书接受对应的IP。对于Kubeadm而言，apiserver-advertise-address选项可以指定使用的IP。

# kubelet
kubelet向apiserver注册K8s的node节点，这其中包括node使用的InternalIP。为了让InternalIP使用指定网络的IP，我们需要给Kubelet加上--node-ip参数。

# flannel
flannel默认使用node的第一块网卡地址作为overlay数据包的出口。为了修改flannel的使用特定IP，我们可以修改flannel的yaml文件，增加--iface选项指定flannel使用的网卡或者IP，例如--iface=ens1f1，这个选项可以指定多次，flannel将依次尝试，直到找到正确的选项。

# dns
当coredns解析失败之后，dns将查询物理机配置的dns服务器，dns查询流量将继续走到其他网卡。需要修改物理机的dns配置。