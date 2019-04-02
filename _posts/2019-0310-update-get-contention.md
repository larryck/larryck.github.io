---
title: k8s go client update/get contention 
description: 
categories:
- k8s
tags:
- bug
---

Currently, we find that when we sucessfully update a object to k8s and get it instantly, we might get returned by the previous unchanged object.

In k8s, the object update process looks like below:

	func (csu *objectUpdater) UpdateLksclusterStatus(cluster *v1alpha1.Lkscluster, status *v1alpha1.LksclusterStatus) error {
	        return retry.RetryOnConflict(retry.DefaultRetry, func() error {
	                cluster.Status = *status
	                _, updateErr := csu.client.LksclusterV1alpha1().Lksclusters(cluster.Namespace).Update(cluster)
	                if updateErr == nil {
				return nil
	                }
	
	                updated, err := csu.clusterLister.Lksclusters(cluster.Namespace).Get(cluster.Name)
	                if err != nil {
	                        glog.Errorf("Error getting updated Cluster %s/%s: %v", cluster.Namespace, cluster.Name, err)
	                        return err
	                }
	
	                // Copy the Cluster so we don't mutate the cache.
	                cluster = updated.DeepCopy()
	                return updateErr
	        })
	}

Here we add some delay operations to wait for the the ojbect sync:

	(csu *objectUpdater) UpdateLksclusterStatus(cluster *v1alpha1.Lkscluster, status *v1alpha1.LksclusterStatus) error {
	        return retry.RetryOnConflict(retry.DefaultRetry, func() error {
	                cluster.Status = *status
	                _, updateErr := csu.client.LksclusterV1alpha1().Lksclusters(cluster.Namespace).Update(cluster)
	                if updateErr == nil {
	                        // wait cache synced to make sure that we get the synchronized
	
	                        for i:=0; i<WaitCacheSyncTimes; i++ {
	                                updated, err := csu.clusterLister.Lksclusters(cluster.Namespace).Get(cluster.Name)
	                                if err != nil {
	                                        glog.Errorf("Sync Error getting updated Cluster %s/%s: %v", cluster.Namespace, cluster.Name, err)
	                                        return err
	                                }
	                                if updated.Status.Phase !=  status.Phase {
	                                        time.Sleep(100 * time.Millisecond)
	                                } else {
	                                        return nil
	                                }
	                        }
	                        glog.Errorf("Sync Error sync Cluster cache %s/%s", cluster.Namespace, cluster.Name)
	                        return fmt.Errorf("Sync Error sync Cluster cache %s/%s", cluster.Namespace, cluster.Name)
	                }
	
	                updated, err := csu.clusterLister.Lksclusters(cluster.Namespace).Get(cluster.Name)
	                if err != nil {
	                        glog.Errorf("Error getting updated Cluster %s/%s: %v", cluster.Namespace, cluster.Name, err)
	                        return err
	                }
	
	                // Copy the Cluster so we don't mutate the cache.
	                cluster = updated.DeepCopy()
	                return updateErr
	        })
	}

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
