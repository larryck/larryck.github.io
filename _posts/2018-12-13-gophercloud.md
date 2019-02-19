---
title: gophercloud中使用token认证
description: 
categories:
- k8s
tags:
- openstack
---

gophercloud是一个成熟的openstack go client，包括kubernetes官方cloudprovider在内的很多开源大项目都用到了gophercloud。LKS也不例外，更多关于LKS的设计可以查看之前的文章[LKS Kubernetes Service设计解密](https://larryck.github.io/k8s/2018/12/12/lks-k8s-service/)。在LKS中，我们的一个特殊需求是，因为与openstack horizon相融合，gophercloud client需要使用token认证的方式与OpenStack进行交互。这篇文章是LKS系列文章的一部分，在本文中，我们将探讨gophercloud的token认证方式。

# user/passwd认证
在[gophercloud的官方github](https://github.com/gophercloud/gophercloud)中列出了gophercloud client使用用户名/密码认证的方式与Openstack交互的方式。关键代码如下所示：

    // Option 1: Pass in the values yourself
    opts := gophercloud.AuthOptions{
        IdentityEndpoint: "https://openstack.example.com:5000/v2.0",
        Username: "{username}",
        Password: "{password}",
    }

    provider, err := openstack.AuthenticatedClient(opts)

# token认证
然而，在实际应用中，用户对于提交登陆密码等敏感信息具有很大的顾虑，因而LKS设计了基于token的认证方式。但是，gophercloud官方并没有相关文档介绍client如何使用token进行认证。通过深度的代码分析，我们发现gophercloud是支持token认证的，使用示例如下所示：

    // Option 3: auth with token
    opts := gophercloud.AuthOptions{
        IdentityEndpoint: "https://openstack.example.com:5000/v2.0",
        TokenID: "{tokenid}",
        TenantID: "{tenantid}",
    }

    provider, err := openstack.AuthenticatedClient(opts)

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)