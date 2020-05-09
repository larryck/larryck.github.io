---
title:  lksapiserver设计揭秘
description: 
categories:
- k8s
tags:
- design
---

# 问题
如何设计lksapiserver满足用户体验一致性以及底层平台无关？

# 方案
lksapiserver是lks架构优化的关键组件之一。 lksapiserver负责承接用户交互，同时与LKS系统中各个Operator交互，完成用户指定的工作。

lksapiserver需要满足以下两个特性：
- 用户体验一致性： LKS支持Dashboard以及cmdline工具两种交互方式，两种交互方式的认证流程需要一致；lksapiserver需要隔离用户交互中的平台相关部分，保证在不同底层平台在上层的用户体验上相同；
- lksapiserver作为向上提供RESTful api访问方式的主体，可以将LKS的平台能力向上提供。

## 用户体验一致性
LKS的用户交互支持dashboard以及cmdline交互，这两种方式在需求上有所不同：使用命令行交互时，工具可以将特定位置的配置文件用作认证，保证每次操作的权限控制；使用UI时，用户需要登录，并且token需要有超时机制。

LKS的做法是LKS统一使用lksconfig文件认证，这个文件包含用户的IaaS平台登录信息以及证书。使用命令行时，这个文件放在~/.kube下；使用UI时，用户选择这个文件登录。lksapiserver将通过登录信息以及证书验证用户的身份，并且生成相应的token。通过这种方式统一两种交互方式的认证流程。

在平台无关性实现上，lksapiserver提供了lksapiserver-authi接口，实现封装了所有和平台相关的操作实现，从而隔离了上层交互与平台相关接口的交互。关于lksapiserver-authi的信息可以通过之前的文章了解。

## RESTful api
lksapiserver服务通过http server对外提供，遵循标准的REST交互。在访问lksapiserver api之前，用户需要通过lksconfig认证文件获得正确完整的认证token。