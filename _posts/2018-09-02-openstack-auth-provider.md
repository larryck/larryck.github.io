---
title: openstack auth provider
description: 
categories:
- k8s
tags:
- auth
---

A openstack auth provide has been merged into the upstream k8s repository, which uses the openstack environmental variables as information needed by k8s auth plugins.

# auth
The k8s-keystone-auth service creates a token by authenticate with keystone, which k8s will use as bearer token in all actions then.

	req.Header.Set("Authorization", "Bearer "+token)

More info can be found [here](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-keystone-webhook-authenticator-and-authorizer.md).
