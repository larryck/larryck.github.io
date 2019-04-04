---
title: bug caused by for iteration in go
description: 
categories:
- k8s
tags:
- bug
---

Here is the code:

        for _, cn := range checked {
                if cn.Status.Phase == v1alpha1.LksnodeVMReady {
                        go m.deployK8sNode(lc, &cn)
                }
        }

We find that in for iteration, the cn variable reference is not changed. Suppose we have two **cn**s in **checked**, this means that when the second go thread starts with cn set to the second object, the first thread's cn will also changes, which ends up into mess. The solution is to pass a blank new cn to each thread. 

        for _, cn := range checked {
                if cn.Status.Phase == v1alpha1.LksnodeVMReady {
                        newcn := cn
                        go m.deployK8sNode(lc, &newcn)
                }
        }


---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
