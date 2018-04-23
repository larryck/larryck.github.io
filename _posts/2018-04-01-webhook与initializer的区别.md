---
title: webhook与initializer的区别 
description: 
categories:
- k8s
tags:
- features
---

Webhook及initializer都是k8s dynamic admission controller的一部分，为pods在调度前的检查、修改等功能提供一些操作hooks。k8s文档对这两个功能的描述不太详尽，本文通过分析两个示例源码记录了webhook与initializer的使用方式。

# webhook
webhook源码示例来自于[example-webhook-admission-controller](https://github.com/caesarxuchao/example-webhook-admission-controller)。从代码中分析我们可以得到一下几点：

1、 Webhook是一个http server，从根请求提供服务。k8s主动向webhook发起请求。
```
    http.HandleFunc("/", serve)
```
2、Webhook处理请求是一个AdmissionReview的json化字符串，从Raw属性中可以获得pod对象。
```
    json.Unmarshal(data, &ar)
    ...
    raw := ar.Spec.Object.Raw
    pod := v1.Pod{}
    json.Unmarshal(raw, &pod)
```
3、Webhook的的处理结果是一个AdmissionReviewStatus对象，通过Allowed属性标记处理结果。
```
reviewStatus := v1alpha1.AdmissionReviewStatus{}
reviewStatus.Allowed = true
```
4、Webhook返回结果是设置了Status属性的AdmissionReview json化对象。
```
    ar := v1alpha1.AdmissionReview{
        Status: *reviewStatus,
    }
    resp, err := json.Marshal(ar)
```

# initializers
initializer代码示例来自于[kubernetes-initializer-tutorial](https://github.com/kelseyhightower/kubernetes-initializer-tutorial)。从代码分析我们可以得出以下几点：

1、initializer通过带IncludeUninitialized = true
的listwatch接口监听k8s对象：
```
    includeUninitializedWatchlist := &cache.ListWatch{
        ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
            options.IncludeUninitialized = true
            return watchlist.List(options)
        },
        WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
            options.IncludeUninitialized = true
            return watchlist.Watch(options)
        },
    }
```
2、initializer的获取方式，位于ObjectMeta的Pending列表
```
pendingInitializers := deployment.ObjectMeta.GetInitializers().Pending
```
3、initializer依次处理：处理完之后将当前initilizer取下并更新对应的k8s对象
```
            // Remove self from the list of pending Initializers while preserving ordering.
            if len(pendingInitializers) == 1 {
                initializedDeployment.ObjectMeta.Initializers = nil
            } else {
                initializedDeployment.ObjectMeta.Initializers.Pending = append(pendingInitializers[:0], pendingInitializers[1:]...)
            }
```

```
            patchBytes, err := strategicpatch.CreateTwoWayMergePatch(oldData, newData, v1beta1.Deployment{})
            if err != nil {
                return err
            }

            _, err = clientset.AppsV1beta1().Deployments(deployment.Namespace).Patch(deployment.Name, types.StrategicMergePatchType, patchBytes)
            if err != nil {
                return err
            }
```



