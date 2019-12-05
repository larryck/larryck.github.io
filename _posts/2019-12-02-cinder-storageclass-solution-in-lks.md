---
title: cinder storageclass solution in LKS
description: 
categories:
- k8s
- storage
tags:
- policy
---

# 问题
Openstack cloudprovider中已经有关于cinder storageclass的实现。对于LKS而言，为了保证用户体验的一致性，我们需要重新设计原始的cinder-csi-plugin，让cinder-csi-plugin能够在LKS K8s服务中为不同租户的K8s集群提供服务。本文中，我们介绍Lkscindercsi v1的设计。

目前LKS提供的K8s版本为V1.13，因而本文中对csi架构的讨论集中在csiv0。

# 单点登录
作为一个完善的产品，我们要求用户使用OpenStack账户登录之后，使用LKS的所有服务不需要再重新认证。

这个需求对于cinder-csi-plugin更加具有挑战性。因为在storageclass创建之后，之后的每一次PV创建都需要通过OpenStack认证。cinder-csi-plugin默认使用用户名/密码进行认证，然而LKS服务并不记录用户的用户名/密码，这种方案不可行。如果使用token方案，在关机及意外断电等极端情况会给方案的运维带来很大复杂性，因而这个方案也不考虑。

我们的选择是通过admin用户完成：admin用户认证之后使用在用户的tanent下为PV创建相应的块设备，并且将块设备绑定到对应云主机。为了保证admin用户信息的安全性，我们需要将块设备的创建及绑定部分逻辑隔离到普通用户无法访问的集群中。

# cinder-csi-plugin 
