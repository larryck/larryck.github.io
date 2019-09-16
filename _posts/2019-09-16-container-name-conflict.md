---
title:  kubernetes pods cannot start after reboot due to container name conflict
description: 
categories:
- k8s
tags:
- bug
---

# 错误现象
在机房断电恢复LKS（我们自己开发的[智能Kubernetes服务](https://larryck.github.io/k8s/2018/12/12/lks-k8s-service/)）的过程中，我发现在Kubernetes集群重启之后有一些Pod一直处于Error状态。通过查询错误原因，我发现了如下错误信息：

```
failed to create a sandbox for pod "duke-0": Error response from daemon: Conflict. The container name "/k8s_POD_duke-0_5114e2-duke_2f8d05f5-ae84-11e9-9652-fa163eca445b_3" is already in use by container "466e7ecb2762cb6a12e8bd0b8a80aca90991c762d8364e236992380ecc6d4656". You have to remove (or rename) that container to be able to reuse that name.
```

# 问题分析
社区中已经发现了在今年七月份已经发现了[这个问题](https://github.com/kubernetes/kubernetes/pull/79623)。 在本文中，我将总结并在分析LKS使用的版本V1.13.1中的问题。

问题的起因是在某些情况下docker将进入一种不一致的内部状态。这种不一致状态的一个现象是某些错误的容器在docker中已经存在，而docker进程并未处理。当节点重启等情况下kubelet尝试创建docker容器时docker将返回命名冲突，即创建的容器使用的名称在已经存在。这时将返回我们开篇提到的错误。

Kubernetes中使用了一种简单的方法处理这个问题，即当创建SandBox容器出现这个错误时，Kubernetes将使用正则匹配获取已经存在的容器的ID，并删除已存在的容器，再次尝试创建。提取容器ID的正则表达式如下所示。
```
conflictRE = regexp.MustCompile(`Conflict. (?:.)+ is already in use by container ([0-9a-z]+)`)
```
这部分代码逻辑位于V1.13.1的kubernetes/pkg/kubelet/dockershim/helpers.go文件中。

[这篇文章对这个问题](https://github.com/kubernetes/kubernetes/pull/36768)做了详细介绍。

然而，随着docker版本的升级，docker代码中对命名冲突的错误信息做了一些小变更：在错误信息容器ID字段使用%q输出，这就导致容器ID字段自动加上了双引号，可以看[这篇文章](https://github.com/moby/moby/pull/27510)的描述。我核实了docker 1806、1809、1903的代码，都已经使用这种输出格式。

**加上双引号后，上述conflictRE正则匹配将失败，导致Kubernetes的删除已存容器机制失效，最终造成了Pod一直处于Error状态。**

问题的解决也非常简单，即将上面的正则表达式换成如下：
```
conflictRE = regexp.MustCompile(`Conflict. (?:.)+ is already in use by container \"?([0-9a-z]+)\"?`)
```

# 问题影响范围
我核实Kubernetes的代码实现，这个Bug在Kubernetes V1.16.0-alpha.2版本开始修复，也就是说Kubernetes V1.16.0-alpha.1版本之前的所有版本都这个问题都存在。然而，我也注意到，社区对于V1.13以后的release版本都已经做了修复，但是V1.12之前的版本这个问题依然存在。

