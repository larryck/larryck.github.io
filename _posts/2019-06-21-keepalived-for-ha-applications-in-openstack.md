---
title:  keepalived for ha applications on openstack
description: 
categories:
- openstack
tags:
- policy
---

为了达到HA的目的，越来越多的应用选择多备份执行，并对外暴露一个访问入口。然而，在OpenStack平台上，达到应用高可用的方式有很多种，这里我们探讨Keepalived访问。

# LBaaS
先说OpenStack原生支持的LBaaS服务。LBaaS可以在某个子网中创建一个LB，指定监视器规则，并关联到后端的资源池，从而LB可以将前端的访问流量转发到后端。

LBaaS基于HAProxy实现，本身是一个LB的解决方案，大家也都很熟悉。然而，LBaaS存在的一个问题是需要绑定到端口，即对于每一个监听的端口都需要创建一个新的监视器，并绑定到新的后端资源池。LBaaS目前不支持端口区域配置。

# Keepalived
为了支持多端口的HA解决方案，我们选择了Keepalived。

一个问题是使用Keepalived时，在系统中会存在OpenStack不识别Ip/Mac对应的网络包，这些网络包将会被OpenStack的安全策略丢弃。因为，为了使用Keepalived我们首先需要让OpenStack识别这些网络包。

为了让虚机接受具有vip/mac对的网络包，我们需要为虚机的网络端口配置--allowed-address-pair属性，添加对vip的接受策略。具体的步骤如下：
- 创建vip对应的port，获取私有网络中vip对应的地址。这个port对应的是在私有网络中vip的端口，用来绑定vip；
- 创建虚拟机使用的port，并且通过--allowed-address-pair使得这个port能够接受vip网络包；
- 使用上述port创建虚机，并在security policy配置访问区域；
- 在虚机中配置Keepalived；
- 为vip port绑定公网IP；

之后，应用可以通过vip对应的公网IP访问Keepalived后端的虚机中的应用。