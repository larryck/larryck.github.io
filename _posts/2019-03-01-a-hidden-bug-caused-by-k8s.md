---
title:  hidden bug caused by k8s
description: 
categories:
- k8s
tags:
- bug
---

We know that k8s will inject existing service env variables into pods create later. For example, if we have a service named 'dm' existing in k8s, env variables like DM_PORT/DM_SERVICE_HOST/DM_SERVICE_PORT will be set for later creating pods. This might cause some hidden problems if applications running in the pod using env variables with the same name with those preseting variables.  The problem we met is a database named KB. We containerize KB to port it to our k8s platform, but we got the "port must be integer" failure in k8s but it ok if we just run it with docker. The reason is that KB use the KB_PORT env variable as port of the database but k8s inject the same env varible.

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)