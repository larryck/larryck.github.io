---
title: k8s code-generator generate-groups
description: 
categories:
- k8s
tags:
- statefulset
---

When we try to conduct stateful apps with CustomResourceDefinition(crd for short), a really tedious thing is to build up clientset/informers/listers for the CRD. Fortunately, the k8s project has provided a code-generating process for us, which is the generate-groups function in code-generator project.


     # generate-groups generates everything for a project with external types only, e.g. a project based
     # on CustomResourceDefinitions.


    Usage: $(basename $0) <generators> <output-package> <apis-package> <groups-versions> ...
    
      <generators>the generators comma separated to run (deepcopy,defaulter,client,lister,informer) or "all".
      <output-package>the output package name (e.g. github.com/example/project/pkg/generated).
      <apis-package>  the external types dir (e.g. github.com/example/api or github.com/example/project/pkg/apis).
      <groups-versions>   the groups and their versions in the format "groupA:v1,v2 groupB:v1 groupC:v2", relative
      to <api-package>.
      ... arbitrary flags passed to all generator binaries.
    

The first step to generate a clientset for a crd resource is to **define crd structs**, which is the **apis-package** parameter of generate-groups. Some tags in the api-package give the generate-groups tool indications on how to manipulate each resource, for example:
    
    // +genclient
    // +genclient:noStatus
    // +resourceName=myresoucename
    // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
	type MyResource struct {

From the tags above, the generate-groups tool know that it knows to generate clientset for **MyResource**, and the resource name of **MyResource** is **myresourcename**.
    
Here is an example, 

    k8s.io/code-generator/generate-groups.sh all github.com/example/project/pkg/client github.com/example/project/pkg/apis "foo:v1 bar:v1alpha1,v1beta1"

After that, the github.com/example/project/pkg/client directory will be generated, which containers clientset/informers/listers packages.
