---
title: why istio not collect application metrics
description: 
categories:
- istio
- envoy
- k8s
tags:
- policy
---

# 问题现象
我们使用kube-inject -f depl.yaml注入istio-proxy容器。应用启动之后我们发现，Prometheus不能获得应用的性能指标，包括mixer的性能指标以及envoy的性能指标。

# mixer性能指标
为了使用istio提供的应用级性能指标收集机制，istio要求对service的端口命名，以<protocol>[-<suffix>]的方式。对于http服务而言，如果端口名不是以http开始的合规名称，istio将其作为tcp服务处理。

# envoy性能指标
我们从Prometheus中我们只能看到cluster.xds-grpc服务的信息，最重要的当前proxy所属应用pod的性能信息。

## 默认最小信息收集
默认情况下，istio将配置envoy收集最少的的性能数据。默认的收集主体包括：
- cluster_manager
- listener_manager
- http_mixer_filter
- tcp_mixer_filter
- server
- cluster.xds-grpc

如果要包含新的指标，需要使用sidecar.istio.io/statsInclusionxxx annotations增加收集主体。

## deploy增加annotations不工作
我们在deploy的annotation中增加了相应sidecar.istio.io/statsInclusionxxx内容，但是从Prometheus的并没有发现相应的指标。也就是说，annotations并不工作。

研究发现，sidecar.istio.io/statsInclusionxxx annotation并没有配置到istio-proxy容器中。sidecar.istio.io/statsInclusionxxx配置项只有配置到ISTIO_METAJSON_ANNOTATIONS环境变量中才能生效。

那么问题是：kube-inject为什么不将deploy中的sidecar.istio.io/statsInclusionxxx annotation更新到istio-proxy容器中？