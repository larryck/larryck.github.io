---
title: DNS-Ingress为k8s应用提供基于域名访问方式
description: Ingress为k8s提供了除Node Port以及Service IP之外的另一种应用访问方式:它为执行在k8s中的应用提供了一个http url到pod ip+port的映射，从而用户可以k8s外部直接通过http请求访问执行在k8s中的应用。
categories:
- k8s
tags:
- network
---

Ingress为k8s提供了除Node Port以及Service IP之外的另一种应用访问方式:它为执行在k8s中的应用提供了一个http url到pod ip+port的映射，从而用户可以k8s外部直接通过http请求访问执行在k8s中的应用。
Ingress提供了host+path指定http访问路径，host对应nginx的virtual server，path指定virtual server下的路径。Ingress对象创建如下所示：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard
  namespace: 25179ff49b014e8fb31d38aaa193ec44
spec:
  rules:
     - host: 25179ff49b014e8fb31d38aaa193ec44.k8s.com
       http:
        paths:
        - backend:
            serviceName: dashboard
            servicePort: 80
```

理所当然的想法是，当我们通过http访问25179ff49b014e8fb31d38aaa193ec44.k8s.com地址时，请求将定向到dashboard service的80端口。
然而，事情并没有想象中的简单。当我们通过http访问25179ff49b014e8fb31d38aaa193ec44.k8s.com时，首先需要通过DNS解析这个地址。
Ingress Controller提供了25179ff49b014e8fb31d38aaa193ec44.k8s.com到k8s内部pod ip+port的转发，然而这个请求如何转发到Ingress Controller？
我们先看Ingress Controller的部署文件：
```
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
          
```
IngressController使用**Host port**的方式映射了http/https端口，当http/https请求转发到IngressController所在节点的80/443端口时，请求将被nginx controller处理。
那么，剩下的问题就是外部的http/https请求如何转发到IngressController所在节点。
此时，我们需要使用一个DNS服务器，例如named。DNS服务器可以指定一个泛域名，例如k8s.com，所有以k8s.com结尾的请求都将解析到IngressController所在节点。而IngressController将识别25179ff49b014e8fb31d38aaa193ec44.k8s.com是用户指定的host，从而完成流量转发。
如此，从k8s外部使用域名访问pod中执行应用的通道就已经打通了。

###ingress对象与nginx.conf的映射
当我们使用如下模板创建ingress对象时，
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: authserver-ingress-path
  namespace: kube-system
spec:
  rules:
     - host: testk8spath.com
       http:
        paths:
        - path: /login
          backend:
            serviceName: authserver
            servicePort: 80
        - path: /auth
          backend:
            serviceName: authserver
            servicePort: 5012
```
ingress controller的nginx.conf将创建一些新的以testk8spath.com为名的server，并且对/login、/auth、以及默认路径/都指定相应的proxy_pass。如下所示：
```
   server {
        server_name testk8spath.com;
        ........
        location /login {
            set $proxy_upstream_name "kube-system-authserver-80";
        ........
            proxy_pass http://kube-system-authserver-80;
        }
        location /auth {
            set $proxy_upstream_name "kube-system-authserver-5012";
        ........
            proxy_pass http://kube-system-authserver-5012;
        }
        location / {
            set $proxy_upstream_name "upstream-default-backend";
        ........
            proxy_pass http://upstream-default-backend;
        }
        
```
其中，以*kube-system-authserver-80*为例：
```
    upstream kube-system-authserver-80 {
        # Load balance algorithm; empty for round robin, which is the default
        least_conn;
        server 10.233.xx.xx:80 max_fails=0 fail_timeout=0;
    }
```

值得注意的是，10.233.xx.xx使用的是service对应的**pod的ip**，如果service下有多个pod，则nginx为这些pod做负载均衡。


