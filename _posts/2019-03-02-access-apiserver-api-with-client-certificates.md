---
title:  access apiserver api with client certificates
description: 
categories:
- k8s
tags:
- security
---

kubectl accesses apiserver with a config file, which is placed in ~/.kube commonly. Client config file has information like user name, apiserver address, certificate-authority-data, client-certificate-data, and client-key-data, all this information and RBAC rules in k8s setup the authentication and authorition environment for kubectl client.

Kubectl is commonly used for commandline tools, but for applications that access apiserver with REST interface, certificate files, include ca, cert, cert, are required. 

We can extracted certificate files from kubectl config.  The three domains, certificate-authority-data, client-certificate-data, and client-key-data, contains all information we need, but they are base64 encrypted. We need to decode them before pass them to http requests. Below is how we convert config file to certificates:

```
echo $certificate-authority-data | base64 -d > ca.pem
... > cert.pem
... > key.pem

curl --cert ./cert.pem --key ./key.pem --cacert ./ca.pem https://[apiserver]/api/v1/pods
```

# access apiserver with proxy and token
We can proxy apiserver to a http endpoint so that applications visit k8s with http protocol. It runs like:

```
kubectl proxy --port=8080 &
curl http://localhost:8080/api/
```

Also, we can extract the Bearer token to use for REST access

```
kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='default')].data.token}"|base64 -d
```

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)
