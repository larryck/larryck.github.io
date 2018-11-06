---
title: k8s OwnerReference
description: 
categories:
- k8s
tags:
- policy
---

In k8s, every dependent object has a metadata.ownerReferences field that points to the owning object.

In some circumstances, Kubernetes sets the value of ownerReference automatically. For example, when you create a ReplicaSet, Kubernetes automatically sets the ownerReference field of each Pod in the ReplicaSet.
Or, you can also specify relationships between owners and dependents by manually setting the ownerReference field.

# Resouce Deletion
When you delete an object, you can specify whether the object’s dependents are also deleted automatically. Deleting dependents automatically is called **cascading deletion**. 

To control the cascading deletion policy, set the propagationPolicy field on the deleteOptions argument when deleting an Object. Possible values include “Orphan”, “Foreground”, or “Background”. There are two modes of cascading deletion: background and foreground.  If you delete an object without deleting its dependents automatically, the dependents are said to be **orphaned**.

In foreground cascading deletion, the root object first enters a “deletion in progress” state. Once the garbage collector has deleted all “blocking” dependents (objects with ownerReference.blockOwnerDeletion=true), it deletes the owner object. In background cascading deletion, Kubernetes deletes the owner object immediately and the garbage collector then deletes the dependents in the background.


In some versions, unless you specify otherwise, dependent objects are orphaned by default, while in others, dependent objects are deleted by default.

kubectl also supports cascading deletion. To delete dependents automatically using kubectl, set **--cascade** to true. To orphan dependents, set --cascade to false. The default value for --cascade is true.
