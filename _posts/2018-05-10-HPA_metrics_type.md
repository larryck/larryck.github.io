---
title: k8s HPA metrics type
description: 
categories:
- k8s
tags:
- cluster
---

在分析HPA代码过程中遇到了metricSpec.Type的三个值：ObjectMetricSourceType、PodsMetricSourceType、ResourceMetricSourceType，看着有些不清晰。在本文中记录一下HPA支持的metric类型。分析的k8s版本为v1.6.7。

# ResourceMetricSourceType
这个类型代表资源使用类型，包含targetAverageValue以及targetAverageUtilization两种衡量标准。如果使用**targetAverageValue**，HPAController将获取selector对应pod的Container最新资源使用情况，并根据这个值与targetAverageValue计算得到目标副本数；如果使用**targetAverageUtilization**，HPAController将获取selector对应pod的Container最新资源使用情况，并将之与container的资源request值计算pod的资源使用率，与targetAverageUtilization计算得到目标副本数。这个过程中HPAController将会考虑尚未Ready以及Missing的pod情况。进一步的分析可以查看另一篇文章[HPA逻辑分析](https://larryck.github.io/k8s/2018/05/11/HPA策略/)。ResourceMetricSourceType目前仅支持CPU及Memory两种性能指标。

# PodsMetricSourceType
这个类型与使用targetAverageValue标准的ResourceMetricSourceType.类似。不同于ResourceMetricSourceType的是，PodsMetricSourceType使用的是Heapster的api接口组（见[Heapster访问接口](https://larryck.github.io/k8s/2018/05/13/Heapster访问接口/)），能够使用指标类型多于ResourceMetricSourceType类型。

# ObjectMetricSourceType
这个类型使用k8s集群中的对象数作为衡量标准。在v1.6.7使用heapster作为数据源（默认）时，暂时不支持这个类型。不过，k8s的官方文档中的示例使用这个类型衡量Ingress的请求数。

# 自定义类型
从k8s的官方文档中可以看到，HPAController已经支持使用外部数据指标动态调整pod的数量。我们将在以后再研究这个特性。


