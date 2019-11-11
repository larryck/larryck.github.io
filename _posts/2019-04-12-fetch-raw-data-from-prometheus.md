---
title: How to fetch raw data from prometheus
description: 
categories:
- k8s
- prometheus
tags:
- policy
---

# 问题描述
我们都知道Prometheus提供graph接口，可以根据需要抽取数据。然而，当我们对数据处理有疑问时，如何获取Prometheus中的原始数据？

# query_range

Prometheus提供query_range api查询原始指标，下面是使用示范。更多的信息可见[Prometheus官方文档](https://prometheus.io/docs/prometheus/latest/querying/api/)。

```
http://localhost:9090/api/v1/query_range?query=envoy_cluster_upstream_rq_time_bucket&start=1573454400&end=1573454430&step=15s

http://localhost:9090/api/v1/query_range?query=envoy_cluster_upstream_rq_time_bucket{pod_name="xxx-1-68958f4f84-smhxs"}&start=1573454400&end=1573454430&step=15s


http://localhost:9090/api/v1/query_range?query=envoy_cluster_upstream_rq_time_bucket{pod_name="xxx-1-68958f4f84-smhxs",cluster_name="inbound|5001|http|xxx.xxx.svc.cluster.local"}&start=1573454400&end=1573454430&step=15s
```


