---
title:  kernel deadlock caused by kmem limit in k8s
description: 
categories:
- k8s
tags:
- bug
---

# 错误现象
a. 终端重复报警消息: **NMI watchdog: BUG: soft lockup - CPU#xx stuck for 22s!**

b. dmesg提示kernel: **SLUB: Unable to allocate memory on node -1**

c. crash log显示: 
```
SLUB: Unable to allocate memory on node -1 (gfp=0xd0)
  cache: sock_inode_cache(158:8aff71103b350fcc15a20dcad154986160ce07496a9f70fa4aff83e7716b0936), object size: 632, buffer size: 640, default order: 3, min order: 0
  node 0: slabs: 8, objs: 363, free: 0
  node 1: slabs: 17, objs: 867, free: 0
socket: no more sockets
IPv6: Failed to initialize the ICMP6 control socket (err -23)
BUG: unable to handle kernel NULL pointer dereference at 0000000000000020
IP: [<ffffffff894dfb20>] ip6mr_sk_done+0x60/0xf0
.......
[<ffffffff894c984d>] rawv6_close+0x1d/0x40
[<ffffffff8946cf5d>] inet_release+0x7d/0x90
[<ffffffff894a67d0>] inet6_release+0x30/0x40
[<ffffffff893cba35>] sock_release+0x25/0x90
[<ffffffff893d302c>] sk_release_kernel+0x2c/0x60
[<ffffffff894cbbf5>] icmpv6_sk_init+0x125/0x140
[<ffffffff893e0794>] ops_init+0x44/0x150
[<ffffffff893e0943>] setup_net+0xa3/0x160
[<ffffffff893e10e5>] copy_net_ns+0xb5/0x180
[<ffffffff88ebfe89>] create_new_namespaces+0xf9/0x180
[<ffffffff88ec00ca>] unshare_nsproxy_namespaces+0x5a/0xc0
[<ffffffff88e90c13>] SyS_unshare+0x193/0x300
[<ffffffff8951f7d5>] system_call_fastpath+0x1c/0x21
```

操作系统版本是CentOS Linux release 7.5.1804， 内核为3.10.0-862.el7.x86_64，k8s版本是v1.12.3，Docker版本Docker version 18.06.1-ce。

# 问题分析及解决方案
首先我们看到内核提示的错误信息与内存分配有关，我们首先考虑这可能是一个内存泄露问题。同时，当终端不断弹出sock lockup错误信息时，手动stop docker service之后消息立马不再弹出，因而我们也怀疑这是docker造成的问题。

问题先从Linux的内存管理机制说起。在Linux中，内存分为内核内存及用户内存，内核内存设计为专用于Linux内核中系统服务使用，是不可swap的，因而内核内存资源对Linux系统来说是非常宝贵的。然而，正因为这种设计，现实中也存在很多针对内核内存资源的攻击，例如恶意进程可以通过不断地fork新进程从而耗尽系统资源，从而造成系统异常或崩溃，这就是所谓的“fork bomb”。

为了防止出现“fork bomb”，社区中就有提议通过linux内核限制cgroup中的kmem容量使用从而限制恶意进程的行为，即[kernel memory accounting机制](https://lwn.net/Articles/516529/)。当我们通过memory cgroup限制应用的内存使用时，我们不但需要限制应用对用户内存的使用，也需要限制应用对内核内存的使用。kernel memory accounting机制为cgroup的内存限制增加了stack pages（例如新进程创建）、slab pages(SLAB/SLUB分配器使用的内存)、sockets memory pressure、tcp memory pressure等，[内核文档](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)中有详细的描述。kernel memory accounting机制为cgroup提供了memory.kmem.usage_in_bytes配置项用于限制内核内存的使用，这个配置项与memory.limit_in_bytes（总内存限额）配合使用存在下面三种情形，在[lwn文章](https://lwn.net/Articles/516529/)中有详细的介绍：
- memory.kmem.limit_in_bytes == unlimited：这种情形不限制内核内存的使用。
- memory.kmem.limit_in_bytes < memory.limit_in_bytes：在这种情形下，我们详细的指定了内核内存的上限是多少。
- memory.kmem.limit_in_bytes >= memory.limit_in_bytes：在这种情形下，我们只关心包括内核内存及用户内存在内的内存的总使用情况，并不关心内核内存实际使用了多少。在实际使用中，通常是这种情形，我们将memory.kmem.limit_in_bytes设置成大于memory.limit_in_bytes，从而只限制应用的总内存使用。

k8s从1.9版本开始runc默认Enable了kernel memory accounting，即当容器应用设置了memory limit时，容器的memory cgroup中将为memory.kmem.limit_in_bytes设置整形最大值，从而使能内核内存限制机制。我们通过查看memory.kmem.slabinfo的信息即可判断在当前cgroup中kernel memory accounting是否使能，如果未使能，访问这个文件将报输入输出错误。

然而，在4.0以下版本的Linux内核对kernel memory accounting的支持并不完善，社区中逐渐爆出了[slab leak causing a crash when using kmem control group](https://bugzilla.redhat.com/show_bug.cgi?id=1507149)问题，同时runc社区以及Kubernetes社区中也出现了同样的问题，[Enabling kmem accounting can break applications on CentOS7](https://github.com/opencontainers/runc/issues/1725)、[application crash due to k8s 1.9.x open the kernel memory accounting by default](https://github.com/kubernetes/kubernetes/issues/61937)。对于这个问题，在内核低于4版本的系统中给docker容器设置kernel-memory限制时将docker将给出以下提示：
```
You specified a kernel memory limit on a kernel older than 4.0. Kernel memory limits are experimental on older kernels, it won't work as expected and can cause your system to be unstable.
```

这个问题也是我们这次遇到问题的根本原因。kmem的泄露导致系统中的服务内存申请失败，从而造成了内核的soft lock。这个问题的解决方法在[Kernel kmem leak caused by newer versions of Docker](https://support.mesosphere.com/s/article/Critical-Issue-KMEM-MSPH-2018-0006)这篇文章中有很详细的介绍。我们解决的方案是将docker版本升级，版本特性包括Disable kmem accounting in runc on RHEL/CentOS (docker/escalation#614, docker/escalation#692) docker/engine#121。



