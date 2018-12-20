---
title: metrics server分析
description: 
categories:
- k8s
tags:
- monitoring
---

metrics server以heapster为基础，构建了一个独立在k8s apiserver之外的apiserver。本文分析metrics server的结构。

# storage注册
metrics server注册了nodemetrics和podmetrics两种storage，分别对应两种资源的访问方式。其中，nodemetrics没有namespace属性，podmetrics是Namespaced的。

	nodemetricsStorage := nodemetricsstorage.NewStorage(metrics.Resource("nodemetrics"), providers.Node, informers.Nodes().Lister())
	podmetricsStorage := podmetricsstorage.NewStorage(metrics.Resource("podmetrics"), providers.Pod, informers.Pods().Lister())

storage的注册通过kube-aggregator标准的InstallAPIGroup接口注册：

	func InstallStorage(providers *ProviderConfig, informers coreinf.Interface, server *genericapiserver.GenericAPIServer) error {
		info := BuildStorage(providers, informers)
		return server.InstallAPIGroup(&info)
	}

# 数据收集
metrics server默认每隔60 * time.Second * 0.9 收集一次性能数据，并将性能数据写入sink中。

metrics source默认使用kubelet summary source，即从k8s中每个节点的kubelet10250端口获取cadvisor提供的性能数据。

	func (src *summaryMetricsSource) Collect(ctx context.Context) (*sources.MetricsBatch, error) {
		summary, err := func() (*stats.Summary, error) {
			startTime := time.Now()
			defer summaryRequestLatency.WithLabelValues(src.node.Name).Observe(float64(time.Since(startTime)) / float64(time.Second))
			return src.kubeletClient.GetSummary(ctx, src.node.ConnectAddress)
		}()

# 数据存储
metrics server默认使用内存存储，保存本次从summary收集的数据.

	// sinkMetricsProvider is a provider.MetricsProvider that also acts as a sink.MetricSink
	type sinkMetricsProvider struct {
		mu    sync.RWMutex
		nodes map[string]sources.NodeMetricsPoint
		pods  map[apitypes.NamespacedName]sources.PodMetricsPoint
	}

# 数据获取
metrics server从metric sink的nodes/pods数据中查找对应的性能数据并返回。以获取node数据为例：
		metricPoint, present := p.nodes[node]
		if !present {
			continue
		}

		timestamps[i] = provider.TimeInfo{
			Timestamp: metricPoint.Timestamp,
			Window:    kubernetesCadvisorWindow,
		}
		resMetrics[i] = corev1.ResourceList{
			corev1.ResourceName(corev1.ResourceCPU):    metricPoint.CpuUsage,
			corev1.ResourceName(corev1.ResourceMemory): metricPoint.MemoryUsage,
		}


总体来说，metrics server的内部数据逻辑与heapster相似，但是也有一些不同。关于[heapster](https://larryck.github.io/k8s/2018/05/13/Heapster访问接口/)/[cadvisor](https://larryck.github.io/k8s/2018/05/20/cadvisor数据采集/)的分析可以看之前的文章。
