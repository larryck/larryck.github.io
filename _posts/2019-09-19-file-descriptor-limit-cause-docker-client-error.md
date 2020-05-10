---
title: file descriptor limitation cause docker client error
description: 
categories:
- k8s
tags:
- bug
---

# 问题
docker ps命令提示“Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?”，systemctl status显示docker service处于active状态。

# 分析
分析dockerd日志显示dockerd进程打开文件描述符太多。通过lsof可以看见，打开文件主要是docker.sock。

OS默认限制每个进程打开文件数为1024。当系统中存在多个工具与docker交互（例如kubelet、监控工具、其他管理工具等）或者当dockerd响应慢时，容易造成dockerd打开文件超过1024。

我们建议系统默认配置将默认文件限制提升一个数量级。