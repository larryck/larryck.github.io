---
title: ingress rule for http/https 
description: 
categories:
- k8s
tags:
- bug
---

# http
The problem we met for http rules is that the http request will switch to https path. After researching for a while, we find that Ingress frequently uses annotations to configure some options depending on the Ingress controller, different Ingress controller support different annotations. Review the documentation for your choice of Ingress controller to learn which annotations are supported.

The controller we use is kubernetes-ingress-controller. By default the controller redirects all requests to HTTPS if TLS is enabled for that ingress. To configure a specific ingress resources, we must use the nginx.ingress.kubernetes.io/ssl-redirect: "false" annotation in the particular resource. Below is a example of http ingress rule:

	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  annotations:
	    nginx.ingress.kubernetes.io/rewrite-target: /$1
	    nginx.ingress.kubernetes.io/ssl-redirect: "false"
	  name: default-kubernetes-nginx-http
	  namespace: default
	spec:
	  rules:
	  - http:
	      paths:
	      - backend:
	          serviceName: kubernetes-nginx
	          servicePort: 80
	        path: /default-kubernetes-nginx-http/?(.*)
	status:
	  loadBalancer: {}

# https
We have found lots of materials tell us that we need to set the **nginx.ingress.kubernetes.io/secure-backends: "true"** annotation to visit https resources, but this is not enough for kubernetes-ingress-controller.  The other annotation which marks the https resource type is **nginx.ingress.kubernetes.io/backend-protocol: HTTPS**. Below is an example of ingress rules for https:

	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  annotations:
	    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
	    nginx.ingress.kubernetes.io/rewrite-target: /$1
	    nginx.ingress.kubernetes.io/secure-backends: "true"
	  name: kube-system-kubernetes-dashboard-https
	  namespace: kube-system
	spec:
	  rules:
	  - http:
	      paths:
	      - backend:
	          serviceName: kubernetes-dashboard
	          servicePort: 443
	        path: /kube-system-kubernetes-dashboard-https/?(.*)
	status:
	  loadBalancer: {}

# rewrite-target
You may have noticed the **nginx.ingress.kubernetes.io/rewrite-target: /$1** and **/?(.\*)** in the path field. The make sures other resources request by the index page also returns correctly.

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
