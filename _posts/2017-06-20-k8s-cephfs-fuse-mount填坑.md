---
title: k8s cephfs fuse mount填坑
description: 
categories:
- k8s
tags:
- storage
---

从1.10版本开始，k8s已经支持使用fuse mount方式挂载cephfs pv。最近我的工作内容涉及到了基于cephfs为k8s提供外部存储，因而做了一些工作使能cephfs storageclass以及新版本中的fuse mount功能。
我所使用的k8s平台基于Arm64集群、银河麒麟OS、k8s 1.6.7以及ceph v10.1.2。为了能够使用fuse mount功能，我将v1.10的相关代码反向移植到了v1.6.7版本。

###cephfs storageclass
```
type StorageClass struct {
    ......
    // provisioner is the driver expected to handle this StorageClass.
    // This is an optionally-prefixed name, like a label key.
    // For example: "kubernetes.io/gce-pd" or "kubernetes.io/aws-ebs".
    // This value may not be empty.
    Provisioner string
```
在k8s中StorageClass拥有一个Provisioner属性用于实现具体volume的分配与回收。
[kubernetes-incubator/external-storage](https://github.com/kubernetes-incubator/external-storage)提供了cephfs provisioner的一种实现。cephfs Provisioner为cephfs StorageClass提供volume的Provision/Delete接口。Provision接口使用cephfsvolumeclient创建一个特定大小的volume，创建一个随机user id，并使用ceph auth caps命令设置该user对这个目录的读写权限。

###v1.10版本中的fuse mount实现
在1.10版本中，fuse mount的实现比较直白，包括以下步骤：

>- checkFuseMount。如果/sbin/mount.fuse.ceph文件存在则可以尝试fuse mount。
>- 执行*ceph-fuse -k key -m monitor mountpoint -r path*命令mount cephfs目录。如果命令执行出错，则回到使用kernel模块mount的方式。

代码结构比较简单，因而整个反向移植的过程比较轻松。

##两个Bug
###ceph mount failed with (22) Invalid argument
kubernetes-incubator/external-storage提供了一个test pod示例用来验证cephfs storageclass的功能。然而，pod启动时出现了volume挂载错误，具体表现为**ceph mount failed with (22) Invalid argument**。
分析v1.10中ceph-fuse的mount命令发现ceph-fuse命令并没有指定mount使用的用户，因而我判断是这个原因导致的mount失败。针对这个错误，我测试了以下情形：
```
root@node1:~/ck_pls_keep# ceph-fuse -k /mnt/ceph.client.tst1.keyring /mnt/test-fuse/ -r /tst1
ceph-fuse[608668]: starting ceph client2018-02-06 11:55:49.190379 7fa0772000 -1 init, newargv = 0x16cdcb710 newargc=11

ceph-fuse[608668]: ceph mount failed with (22) Invalid argument
ceph-fuse[608666]: mount failed: (22) Invalid argument
```
指定用户id之后，这个错误消失。
```
root@node1:~/ck_pls_keep# ceph-fuse --id tst1 -k /mnt/ceph.client.tst1.keyring /mnt/test-fuse/ -r /tst1
ceph-fuse[608691]: starting ceph client
2018-02-06 11:55:55.997967 7f7bb25000 -1 init, newargv = 0x148c19710 newargc=11
ceph-fuse[608691]: starting fuse
```
进而，我们可以认为是缺少id参数导致k8s报了这个错误。
因而我需要更新pkg/volume/cephfs/cephfs.go 文件的execFuseMount函数，增加--id选项。详情可以见我提到kubernetes社区的[issue #59393](https://github.com/kubernetes/kubernetes/issues/59393)。

        src += hosts[i]
        mountArgs := []string{}
        
        //add by tanck
        mountArgs = append(mountArgs, "--id")
        mountArgs = append(mountArgs, cephfsVolume.id)
        
        mountArgs = append(mountArgs, "-k")
        mountArgs = append(mountArgs, keyring_file)

我测试了k8s v1.10.0-alpha.3版本，确认这个问题依旧存在。
###mount failed: (1) Operation not permitted
解决了ceph mount failed with (22) Invalid argument问题之后，test pod测试依旧出现了**mount failed: (1) Operation not permitted**错误。
经测试发现，当cephfs的目录名包含'-'字符时ceph-fuse将报*mount failed: (1) Operation not permitted* 错误，而使用'\_'字符分割则不会报这个错误。而kubernetes-incubator/external-storage创建随机目录名时正好使用'-'分割。因而我更新了*cephfs/cephfs-provisioner.go*文件的Provision函数将share name的'-'字符替换为'_'字符。
```
        // create random share name

        share := fmt.Sprintf("kubernetes-dynamic-pvc-%s", uuid.NewUUID())

        **//add by tanck
        share = strings.Replace(share, "-", "_", -1)**
```
详情可见我提到kubernetes-incubator/external-storage的[issue #589](https://github.com/kubernetes-incubator/external-storage/issues/589)。
解决完这两个问题之后test pod测试cephfs storageclass + fuse mount功能正常。
k8s对cephfs fuse的支持目前还比较初级，很多功能在后续的使用中还需要不断地完善。

