---
title: k8s hostport方式提供应用对外访问-引用
description: The hostPort setting applies to the Kubernetes containers. The container port will be exposed to the external network at **hostIP:hostPort**, where the hostIP is the IP address of the Kubernetes node where the container is running and the hostPort is the port requested by the user.
categories:
- k8s
tags:
- network
---

[引用文章](http://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/)

### hostPort 
The hostPort setting applies to the Kubernetes containers. The container port will be exposed to the external network at **hostIP:hostPort**, where the hostIP is the IP address of the Kubernetes node where the container is running and the hostPort is the port requested by the user. Here comes a sample pod definition:

```
influxdb-hostport.yml
apiVersion: v1
kind: Pod
metadata:
  name: influxdb
spec:
  containers:
    - name: influxdb
      image: influxdb
      ports:
        - containerPort: 8086
          hostPort: 8086
```
The hostPort feature allows to expose a single container port on the host IP. Using the hostPort to expose an application to the outside of the Kubernetes cluster has the same drawbacks as the hostNetwork approach discussed in the previous section. The host IP can change when the container is restarted, two containers using the same hostPort cannot be scheduled on the same node and the usage of the hostPort is considered a privileged operation on OpenShift.

What is the hostPort used for? For example, the nginx based Ingress controller is deployed as a set of containers running on top of Kubernetes. **These containers are configured to use hostPorts 80 and 443 to allow the inbound traffic on these ports from the outside of the Kubernetes cluster.**

