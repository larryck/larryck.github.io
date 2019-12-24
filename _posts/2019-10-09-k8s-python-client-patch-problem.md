---
title: Kubernetes python client patch error
description: 
categories:
- k8s
tags:
- policy
---

# 问题
使用Kubernetes python client执行patch_namespaced_custom_object更新k8s对象时出现错误，错误信息如下：

```
HTTP response body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"the body of the request was in an unknown format - accepted media types include: application/json-patch+json, application/merge-patch+json","reason":"UnsupportedMediaType"
```

# 分析
这个问题在社区中已经有发现。概括地说是python client从9升到10之后支持了两种类型http内容处理使用了新的标准，从而与现有的一些客户端不匹配。更多信息可以查看[这里](https://github.com/kubernetes-client/python/issues/866)。

# 解决方案
我们采用了一个临时方案。修改client/apis/custom_objects_api.py中patch_namespaced_custom_object_with_http_info函数，强制使用'application/merge-patch+json'内容类型。
```
        header_params['Content-Type'] = self.api_client.\
            select_header_content_type(['application/merge-patch+json'])
            #select_header_content_type(['application/json-patch+json', 'application/merge-patch+json'])
```
