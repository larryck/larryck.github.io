---
title: why tcp connections not closed after http requests ends 
description: 
categories:
- istio
- k8s
tags:
- policy
---

在AIOps项目中，我们使用istio-telemetry及istio-proxy收集应用性能指标。

在压力测试过程中我们发现，当前端http请求增加时，我们看到服务的tcp连接数随之上升；但是当http请求下降时，tcp连接数并没有变化。Why?

# 现象

## app container
```
# netstat -plnt | grep ESTABLISHED | grep app | wc -l
5
```

## istio-proxy container
```
# netstat -plnt | grep ESTABLISHED | grep envoy | wc -l
147
```

从metrics信息我们发现，envoy_server_total_connections连接数在150以上，然而应用的实际连接数我们从app container中可以看到只有5，另外147个连接是istio-proxy的的连接。

# 分析
这个现象归结到istio-proxy在服务中提供的connection pool。

istio使用destinationrule限制每个service cluster的行为，包括tcp服务及http服务。对http服务而言，istio提供trafficPolicy.connectionPool.http.maxRequestsPerConnection配置项来限制每个tcp连接处理的http数目。当这个值设置为1时，每个http请求将建立一个新的tcp连接。

这个选项设置之后，我们可以看到tcp连接数随着前端的压力变化。