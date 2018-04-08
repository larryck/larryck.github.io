---
title: k8s externel volume provisioner的工作机制
description: 
categories:
- k8s
tags:
- storage
---

接着之前我们对cephfs StorageClass及fuse mount问题的研究，本文回答几个疑问：
> - StorageClass与Provisioner如何关联？
> - Externel provisioner如何注册到k8s系统中？ 
> - StorageClass volume创建请求如何传递到externel provisioner?

StorageClass对象定义如下所示：
```
// StorageClasses are non-namespaced; the name of the storage class
// according to etcd is in ObjectMeta.Name.
type StorageClass struct {
        metav1.TypeMeta `json:",inline"`
        // Standard object's metadata.
        // More info: http://releases.k8s.io/HEAD/docs/devel/api-conventions.md#metadata
        // +optional
        metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

        // Provisioner indicates the type of the provisioner.
        Provisioner string `json:"provisioner" protobuf:"bytes,2,opt,name=provisioner"`

        // Parameters holds the parameters for the provisioner that should
        // create volumes of this storage class.
        // +optional
        Parameters map[string]string `json:"parameters,omitempty" protobuf:"bytes,3,rep,name=parameters"`
```
其中，Provisioner属性是Provisioner的名称，Parameters属性是传递到Provisioner的参数，Provisioner将使用这些参数创建volume存储。

在k8s中，pv的供应由PersistentVolumeController负责。ProvisionClaimOperation函数负责具体的volume provision。具体的执行流程包括：
- GetPersistentVolumeClaimClass获取PV使用的StorageClass；
- findProvisionablePlugin获得StorageClass使用的Volume Provisioner；
- 调用Provisioner的provision方法生成需要的volume。
上述是volume供应的通用流程，对于外部的volume provisioner，k8s存在另一条路径。如下所示：
```
    if plugin == nil {
        // findProvisionablePlugin returned no error nor plugin.
        // This means that an unknown provisioner is requested. Report an event
        // and wait for the external provisioner
        msg := fmt.Sprintf("waiting for a volume to be created, either by external provisioner %q or manually created by system administrator", storageClass.Provisioner)
        ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.ExternalProvisioning, msg)
        glog.V(3).Infof("provisioning claim %q: %s", claimToClaimKey(claim), msg)
        return
    }
```
如果k8s获取到的provision plugin为空，则认为系统内部并不存在所需要的provisioner，由外部provisioner负责本volume的供应。此时k8s将等待外部provisioner完成工作。

回到**kubernetes-incubator/external-storage**中，每一种volume介质如cephfs、NFS等都由一个ProvisionController负责与k8s交互。ProvisionController创建了针对PV、PVC以及StorageClass Controller的Watcher并分别Watch这三种资源的变化。

PVC创建是整个Volume Provision操作的开端，在此我们以AddClaim为例分析Provisioner与k8s的交互过程。当PVC watcher的Add事件监听handler AddFunc监听到PVC Add事件之后，首先函数将判断当前PVC是否由本Provisioner负责，判断的准则中重要的一项是**PVC使用的StorageClass的Provisioner名称是否与本Provisioner的名称一致**。接下来AddFunc将调用provisionClaimOperation，其主要工作则是调用volume provisioner的provision接口创建volume存储，并在创建成功之后在k8s环境中创建PV对象。之后，PVC到volume的绑定则由k8s完成。下面的代码列出了这个流程中的关键步骤：
```
    // Set ClaimRef and the PV controller will bind and set annBoundByController for us
    volume.Spec.ClaimRef = claimRef
    ctrl.client.CoreV1().PersistentVolumes().Create(volume)
```
至此，针对文章之前提出的疑问我们可以得到如下结论：
1、External provisioner并不注册到k8s系统中，而只是通过provisioner name来识别当前PVC是否由本Provisioner处理。k8s并不直接管理Provisioner对象。
2、External provisioner监听k8s系统中的pv及pvc，若PVC使用的StorageClass基于本provisioner，则为该PVC创建对应的PV。
3、StorageClass可以通过Param向externel provisioner传递参数，用于方便Provisioner进一步与存储系统交互。

