---
title: k8s roles/serviceaccounts/secrets/tokens
description: 
categories:
- k8s
tags:
- security
---

在k8s中， roles/rolebindings用来设置用户的角色，限制用户的操作行为。然而，在k8s中，我们还涉及到serviceaccount/secret/token等几种对象的协同操作，那么他们之间的关系是什么呢？

# 服务认证

## rolebinding
通过rolebinding，我们可以把包括定义了操作权限的role绑定到serviceaccount。

## serviceaccount
A service account provides an identity for processes that run in a Pod.

当k8s中的pod（例如，dashboard pod）访问apiserver时，需要serviceacount提供相应的权限。每个namespace在创建时都会创建一个default serviceaccount，当pod创建时，如果不指定对应的serviceAccountName将默认使用default serviceaccount。当然，用户也可以创建并且指定pod使用自己的serviceaccount。

## secret

Kubernetes automatically creates secrets which contain credentials for accessing the API and it automatically modifies your pods to use this type of secret. 当serviceaccount创建时，k8s将创建对应的secret对象用于指定用于认证的secret。

## token

token是k8s提供的一种认证授权方式，通过token，k8s可以识别当前进程的操作权限。

# 用户认证

## kubectl
使用kubectl命令行进行用户认证时，kubeconfig中可以通过contexts[n].user指定当前使用的用户，并通过users[n].user指定用户使用的认证形式，可以使用户名/密码，或者证书。如下所示

	contexts:
	- context:
	    cluster: mycluster
	    user: admin
	  name: default
	kind: Config
	preferences: {}
	users:
	- name: admin
	  user:
	    client-certificate: /tmp/certs/cert.pem
	    client-key: /tmp/certs/key.pem

## online service
在线服务可以使用用户名/密码进行登录验证，验证成功之后返回相应的Bearer Token。
