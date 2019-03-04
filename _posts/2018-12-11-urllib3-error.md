---
title: kubenerte client/python error with urllib3
description: 
categories:
- k8s
tags:
- openstack
---

I used the kubernetes client to interact with my kuberentes cluster (v1.13.1, installed with kubeadm) and got "MaxRetryError, HTTPSConnectonPool Max retries exceeded with url, Caused by ProtocoError(Connection aborted, error(0, Error)))". I checked apiserver log and found "http: TLS handshake error from : EOF"

The operating system I use is CentOS Linux release 7.3.1611 (Core), and the kubernetes client is installed with "pip install kubernetes" which is Version: 8.0.1. Urllib3 is urllib3-1.24.1.

python command "from kubernetes import client, config" cause "'NoneType' object is not callable" in <function _removeHandlerRef at" error.

I downgraded urllib3 to 1.15.1 and both error disappered.

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
