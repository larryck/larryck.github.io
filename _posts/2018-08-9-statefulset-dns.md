---
title: hostname and dns of statefulset pods
description: 
categories:
- k8s
tags:
- statefulset
---
We know that each pod of statefulset has a ordered hostname. 

Let's say pod *pa* is the first pod of statefulset *ss* in namespace *ns*, then the hostname of pa is **pa-0**, and the dns of pa is **pa-0.ss.ns.svc.cluster.local**.


