---
title: docker hang caused by systemd bug
description: 
categories:
- k8s
tags:
- policy
---

# 问题
LKS中台平台上有个集群在运行了4个月左右出现了有一个node处于NotReady状态。Docker已经处于hang状态，docker启动容器一致处于created状态，docker inspect等部分命令不响应。

# 现象分析
从系统表现而言，我们看到以下现象：
- docker启动容器停留在created状态，docker ps看不到对应的容器。
- docker log停在sandbox创建流程。
- lsns可能看到docker对应的ns尚未创建，ifconfig能够看见一个容器的两个veth。
- system log显示systemd上报错误信息"The maximum number of pending replies per connection has been reached"

从system log入手我们发现，systemd当前处于异常状态。通过docker debug日志我们可以看到，docker容器的创建卡在runc启动过程。结合这两点，我们在社区发现了[systemd stop read and process dbus event of runc cgroup invokes](https://github.com/lnykryn/systemd-rhel/issues/266)这个issue。这个问题中，有人提到，systemd在大量的创建和删除持续五个月左右会出现溢出问题，从而dbus不再响应所有消息。因为systemd不响应runc的namespace创建消息，导致docker一直hang在这个阶段。这也可以理解为什么我们能看到network ns并没有创建，而且能够在系统中看到一个容器的两个veth同时出现。

# 解决方案
当这个问题出现时，systemd已经不响应docker信息，我们需要重启systemd解决这个问题，通过重启OS或者执行systemctl daemon-reexec。这个问题2019年在systemd社区已经有了修复，详见[sd-bus: deal with cookie overruns](https://github.com/systemd/systemd/pull/11818/files)。从patch的代码可以看到，原始的代码中直接使用了++b->cookie造成32位无符号整形越界。patch中增加了越界判断以及空隙查找逻辑。我们需要更新systemd到v242版本之后以规避这个问题。