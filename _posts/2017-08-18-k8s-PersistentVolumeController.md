---
title: k8s PersistentVolumeController
description: 
categories:
- k8s
tags:
- storage
---

*本文将研究PV、PVC的创建及绑定过程。*

### PersistentVolumeController的结构

pv controller(PersistentVolumeController)主要由以下几部分组成：
> - PV、PVC及StorageClass的Lister，主要用watch k8s系统中的对象，跟踪各种对象的变化事件。Lister对应的Informer()注册了pv controller的处理函数，将系统中的对象变化反应到volumeQueue/claimQueue队列中，从而驱动controller的运行
> - volumeQueue/claimQueue，维护处理过程中的PV及PVC

pv controller启动了三个go线程分别用于跟踪PVC、PV以及Resync操作。如下所示：
```
    go wait.Until(ctrl.resync, ctrl.resyncPeriod, stopCh)
    go wait.Until(ctrl.volumeWorker, time.Second, stopCh)
    go wait.Until(ctrl.claimWorker, time.Second, stopCh)
```
这里我们主要关注volume以及claim的处理。volumeWorker从volumeQueue中通过Get方法获取待处理的volume，并判断当前volume是否已经在系统中存在。如果不存在，volumeWorker将调用updateVolume进一步处理"volume added"、"volume updated"、"periodic sync"事件。 主要的工作函数是syncVolume。

claimWorker的工作与volumeWorker类似。首先从claimQueue中通过Get方法获得待处理的claim，如果当前claim在当前系统中不存在则调用updateClaim进一步处理"claim added"、"claim updated"以及"periodic sync"事件。这里，主要的工作函数是syncClaim。


### PVC及PV的创建


### PVC with StorageClass
当我们使用StorageClass创建PVC时，PVC描述文件的VolumeName为空。此时，findBestMatchForClaim函数将为此PVC寻找最合适PV。如果当前PVC是完全新创建的，那么findBestMatchForClaim将无法为当前PVC找到合适的PV。在这种情形下，syncClaim将调用provisionClaim创建相应的PV。

provisionClaim流程我们之前在[k8s external volume provisioner的工作机制]()文章中介绍过。这里简述如下：
> - 根据storageclass的provisioner找到对应的provisioner controller
> - provision controller创建volume对应的存储
> - provisioner在k8s中创建volume对象，设置相应的claimRef

当provisionClaim创建的volume对象加入到k8s中之后，pv controller的VolumeInformer将获得Add事件，对应的EventHandler会将volume对象加入到controller.volumeQueue队列中。

volumeWorker将接手接下来的工作，由syncVolume完成。此时，volume的ClaimRef已经指定，但是因为claim还没有与pv绑定，claim.Spec.VolumeName依旧为空，syncVolume判断当前volume处于等待绑定状态。syncVolume将从volume中获得claim对象，并将claim对象加入到pv controller的claimQueue队列中，交由claimWorker完成后续的绑定工作。如下所示：
```
        } else if claim.Spec.VolumeName == "" {

            // In both cases, the volume is Bound and the claim is Pending.
            // Next syncClaim will fix it. To speed it up, we enqueue the claim
            // into the controller, which results in syncClaim to be called
            // shortly (and in the right worker goroutine).
            // This speeds up binding of provisioned volumes - provisioner saves
            // only the new PV and it expects that next syncClaim will bind the
            // claim to it.
            ctrl.claimQueue.Add(claimToClaimKey(claim))
            return nil
        }
```

claimWorker再次被调用。这次findBestMatchForClaim将能够获得正确的volume。接下来syncClaim将完成PVC到PV的绑定。
```
            // Found a PV for this claim
            // OBSERVATION: pvc is "Pending", pv is "Available"
            glog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q found: %s", claimToClaimKey(claim), volume.Name, getVolumeStatusForLogging(volume))
            if err = ctrl.bind(volume, claim); err != nil {
                // On any error saving the volume or the claim, subsequent
                // syncClaim will finish the binding.
                return err
            }
            // OBSERVATION: claim is "Bound", pv is "Bound"
            return nil
```
bind操作的实现比较简单，主要工作是更新k8s系统中对应的pvc和pv对象的属性及状态，包括以下几个操作：
```
ctrl.bindVolumeToClaim(volume, claim)
ctrl.updateVolumePhase(volume, v1.VolumeBound, "")
ctrl.bindClaimToVolume(claim, volume)
ctrl.updateClaimStatus(claim, v1.ClaimBound, volume)
```

###PVC without StorageClass
不使用StorageClass时，PV和PVC分别创建。整个过程依旧由volumeWorker和claimWorker合作完成。如果首先创建了pvc，syncClaim执行findBestMatchForClaim时将不能获得volume。claim此时将被设为Pending状态，syncClaim在后续获得正确的volume之后将继续完成绑定工作。如下所示：
```
            // Mark the claim as Pending and try to find a match in the next
            // periodic syncClaim
            if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                return err
            }
            return nil
```

获取Volume之后，syncClaim的后续处理流程与使用StorageClass时一致。*值得注意的是，在PVC中，我们可以指定PV的 name，让PVC绑定特定的PV。*

PV的创建过程也有些不同。当用户创建PV之后，pv controller能够收到volume add事件。由于pv的ClaimRef并没有指定，syncVolume将PV设置成VolumeAvailable状态，等待syncClaim进一步绑定。
```
    if volume.Spec.ClaimRef == nil {
        // Volume is unused
        glog.V(4).Infof("synchronizing PersistentVolume[%s]: volume is unused", volume.Name)
        if _, err := ctrl.updateVolumePhase(volume, v1.VolumeAvailable, ""); err != nil {
            // Nothing was saved; we will fall back into the same
            // condition in the next call to this method
            return err
        }
        return nil
    }
```
至此，整个PV及PVC的创建及绑定过程已经清晰。还有一些其他的过程，如这两个对象的更新及删除、delayBinding等其他技术问题，之后将会陆续研究。

