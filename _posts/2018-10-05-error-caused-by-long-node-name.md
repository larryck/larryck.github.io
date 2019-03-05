---
title: k8s provision error caused by long node hostname
description: 
categories:
- k8s
tags:
- provision
---

When provisioning k8s with kubeadm, the process will fail if the hostname is longer than 64 characters, error show "[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object"。 And we can see the node which kubeadm is trying to patch has a name partly cut from the hostname. Patching the wrong node result in timeout error.

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
