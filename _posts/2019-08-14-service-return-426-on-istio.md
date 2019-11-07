---
title: service return error code 426 on istio
description: 
categories:
- k8s
- istio
tags:
- bug
---

# 错误现象
在我们的一个应用中，我们的一个前端应用通过Nginx proxy_pass的方式将流量转移的后端服务。当我们引入istio之后我们发现，REST接口返回了426错误。

# 原因
研究之后我们发现，istio不支持处理http 1.0请求。对于所有的http 1.0请求，envoy将会返回426错误要求升级成http 1.1。

我们的处理方法是在Nginx的proxy_pass中加入**proxy_http_version 1.1**配置将http版本指定为1.1。

