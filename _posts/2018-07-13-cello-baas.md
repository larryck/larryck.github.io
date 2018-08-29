---
title: cello - blockchain as a service
description: 
categories:
- k8s
tags:
- blockchain
---

我最近一直在k8s平台上为hyperledger fabric提供provision及运维自动化支持，期望形成一个block chain as a service方案。因为cello对k8s的支持还不是很成熟，我们花了很多功夫将fabric启动过程与我们自己的k8s平台相结合。这两天我也关注了cello项目的进展。cello是一个很有野心的项目，在BaaS方面囊括了非常完善的使用场景。

# 支持平台
cello是一个master/slave结构，master上执行cello的service，包括db、operator dashboard、user dashboard、watchdog、engine。其中operator dashboard管理底层平台，负责计算平台的管理、区块链的创建等；user dashboard是区块链业务级别的管理，如区块链的组成、智能合约的管理等；watchdog检查服务的健康状态；engine负责具体的平台相关部署工作。

cello的slave是各个平台节点。cello支持的每种平台作为一个节点，例如一个k8s集群作为cello的一个节点。用户可以决定在哪个节点上创建blockchain实例。cello支持的平台包括docker/docker swarm/k8s/vSphere/baremetal等。

# 面向场景
cello支持的场景分为管理员场景及用户场景，运维人员场景包括：

- 添加/删除节点
- 配置节点
- 操作节点
- 创建/删除区块链
- 操作区块链

运维人员dashboard如下图所示![运维人员界面](https://github.com/hyperledger/cello/raw/master/docs/imgs/dashboard_overview.png)


用户场景包括：

- 申请区块链
- 释放区块链
- 上传/删除智能合约
- 调用/查询智能合约等

区块链用户dashboard如下图所示![区块链用户界面](https://github.com/hyperledger/cello/raw/master/docs/imgs/user-dashboard/overview.png)。

# BaaS
整体而言，cello为BaaS提供的支持已经超出了部署与管理层面。更进一步的，cello也与区块链应用级别的功能相绑定，为区块链用户提供应用级别的信息及管理功能。

虽然当前支持的管理功能还比较简单，项目也并没有完全成熟，但我还是很看好cello在区块链领域成为基础方案的潜力。


