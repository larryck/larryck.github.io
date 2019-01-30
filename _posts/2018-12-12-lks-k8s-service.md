---
title: LKS Kubernetes Service设计解密
description: 
categories:
- k8s
tags:
- k8sservice
---

容器服务已经是一个非常火爆的话题，业界的应用开发已经逐渐接受了把容器作为应用发布的新标准。但是，如果容器服务只提供容器接口，用户在使用上会有诸多限制。因而，业内的主流容器云产商纷纷推出了k8s service，将k8s集群作为一个服务提供，例如AWS提供的eks。

LKS是我们设计实现的一个k8s service容器系统。LKS的设计思想是让用户可以自助、自如的使用k8s。所谓自助是说用户创建集群、删除集群、向集群中增添删除节点可以完全自助完成，不需要管理员介入；所谓自如是说当用户在使用的过程中不需要担心破坏整个k8s集群。k8s的控制面由系统管理，用户只使用k8s集群的计算节点，计算节点的错误不会影响到整个k8s集群的可用性。

这个系列文章将详细探究LKS的设计，以及实现过程中的一些思考。 

# 总体架构
LKS使用一个我们称为kube0的k8s集群来管理其他用户的k8s集群。kube0由管理员维护，用户不可访问。当用户创建一个k8s集群时，系统可以使用k8s标准的接口（例如kubectl create -f xxxx.yaml或者通过REST方式等）在kube0中创建一个k8s对象（Lkscluster），删除一个k8s集群也是类似的。更进一步的，我们提供了一个比集群更加细粒度的k8s对象Lksnode来描述用户创建k8s集群的节点，从而用户可以往k8s集群中增加及删除节点。

在kube0中，我们使用一个基于CRD的Operator来管理其他k8s集群对应的对象。我们定义了两个CRD对象，Lkscluster以及Lksnode。两个对象的描述如下：

**Lkscluster**

    apiVersion: ck.io/v1alpha1
    kind: Lkscluster
    metadata:
    name: ""
    namespace: ""
    spec:
        k8s:
            k8sversion: ""
            isha: ""
        plugins:
            plugins:
            - Name: ""
        nodes:
        - number: 
          node:
            vm:
                auth:
                    tenant: ""
                    user: ""
                    password: ""
                userid: ""
                adminPasswd: ""
                networkID: ""
                flavor: ""

其中， spec.k8s.k8sversion描述了需要创建k8s集群使用的版本，spec.k8s.isha指定当前k8s是否需要ha版本。 spec.plugins描述k8s集群启动之后需要安装的k8s插件，如dashboard等。spec.nodes是一个节点数组，每个元素是一个节点replicas。节点replicas描述了多个配置相同的虚拟机实例。整个spec.nodes描述了集群的初始节点情况。 spec.nodes.vm描述了构成k8s集群的各个VM的认证及登陆信息，包括虚机所在的OpenStack用户名称及密码（为了方便用户通过云平台界面自助创建自己的k8s集群，我们也实现了基于token认证的方式，从而不需要暴露用户的密码）、设置虚机创建后的登陆密码、虚机所属的网路及使用的资源模板等。

**Lksnode**

    apiVersion: ck.io/v1alpha1
    kind: Lksnode
    metadata:
      name: ""
      namespace: ""
      labels:
        lksnode.OwnerClusterName: ""
    spec:
      role: ""
      vm:
        auth:
          tenant: ""
          user: ""
          password: ""
        adminPasswd: ""
        networkID: ""
        flavor: ""

Lksnode定义了当Lkscluster集群创建之后向集群中增加的节点描述文件。其中metadata.labels.lksnode.OwnerClusterName描述了该节点所属的Lkscluster集群，spec.role描述当前节点是增加为worker节点还是master节点。

在kube0中，我们可以通过kubectl create -f lkscluster.yaml创建一个k8s集群，并通过kubectl create -f lksnode.yaml向集群中增加一个节点。

LKS目前把OpenStack作为支撑IaaS平台，未来会扩展到其他云平台。

# Lkscluster集群
大体而言，Lkscluster中的节点部署分为两个过程，分别为vm provision以及k8s deployment。Lkscluster集群从定义上区分为初始集群以及新增节点，这主要是为了兼容不同的k8s集群的部署方式。对于初始集群而言，系统等待集群中所有的节点vm provision完成之后才开始部署k8s组件；而对于其他新增的节点，k8s部署在某个节点vm provision完成之后立马进行，不需要在各个新增节点之间同步。除此之外，初始集群还需要完成一些除k8s部署之外的其他工作，我们把初始集群的启动的各个阶段分为：
- initializing
- vm provision: vm开始启动，并等待所有vm启动完成。
- k8s deploying: k8s开始部署，并等待所有k8s的nodes处于ready状态。部署必须的k8s plugins，例如网络插件等。
- plugin install: 用户指定的插件开始部署，并等待所有插件部署完成。
- ready: 初始集群部署完成，用户可以开始使用k8s集群。开始部署通过lksnode新增的节点。

新增Lksnode的启动过程则分为以下几个阶段：
- initializing
- vm provision: vm启动。
- k8s deploying: k8s开始部署。
- k8sready: 节点k8s部署完成。

区分初始集群与后期新增节点带来的另一个好处是我们可以方便地构建managed k8s集群。这就意味着，在启动的过程中我们可以将初始集群部署在某个系统管理的system tenant中，而用户自己创建的work节点则部署在用户自身tanent中。这样，整个k8s集群的控制面由系统管理，用户则只操作计算面。达到的效果是，用户可以自如地使用k8s集群，不需要担心因误操作导致整个集群崩溃。

在LKS中，Lkscluster对初始集群的管理也转化成对应的Lksnode的管理，即对于Lkscluster中描述的每一个节点，都有一个对应的Lksnode对象被创建用于跟踪初始集群中各个节点的状态。

在我们当前的实现中，用户创建的Lkscluster集群基于修改的kubeadm构建。kubeadm提供的基于kubeadm init和kubeadm join的集群构建方法给我们提供了诸多方便，例如，在初始集群的启动过程中，我们只需要等待master节点部署完成即可以使用与lksnode新增节点相同的流程添加初始集群中的其他worker节点。

# Production
将LKS产品化涉及到很多更加细节的考虑，例如可实现性、安全、易用等方面。下面我们列出一些这方面的考虑，我们将在后续的文章中探讨更多的细节设计：
- 如何不需要用户提供密码等敏感信息？基于安全原因，用户不愿意提供敏感信息给LKS，这将成为用户使用LKS的一个后顾之忧。在我们的实现中，我们与云平台融合，使用云平台的访问token作为用户的认证信息，从而做到不需要用户提供敏感信息。
- 用户如何与kube0交互？基于安全原因，用户是不能直接访问kube0的，因而我们需要一个client库作为用户和kube0之间的桥梁。在我们的实现中我们设计了lksclient库。
- 如何限制用户库在kube0中的权限？出于安全原因我们需要给lksclient在kube0相应的角色，这个角色提供了lksclient对Lkscluster以及Lksnode资源的访问权限，并限制了lksclient访问其他资源。在我们的实现中，我们在kube0中为lksclient创建了相应的用户并提供了对应的rbac role。
- 如何让kube0访问不同用户创建的VM？在IaaS云平台中，不同用户的VM可能属于不同的子网并且不一定会绑定public地址，因而kube0无法主动连接在用户tanent中创建的VM。但是，如果我们给kube0所在的VM绑定public地址时，用户tanent中创建的VM则可以主动连接kube0。在我们的实现中，我们为用户tanent中创建的虚机提供了一个lksadmin agent。lksadmin作为systemd管理的服务运行，负责主动连接kube0并与kube0进行交互。

为了更加方便用户自助地使用LKS，我们将LKS集成到了OpenStack的Horizon项目中，作为horizon dashboard project compute中的一个模块。Lkscluster模块在index表中显示当前用户的所有LKS集群以及集群中的所有节点信息，包括节点名称、节点角色、节点状态、k8s版本、集群名称、集群状态以及节点创建时间等。 index表包含三个表级action，分别是创建Lkscluster集群、创建Lksnode节点以及删除Lkscluster集群。index表还包括几个行级action，包括删除Lksnode节点等。

# kube0
kube0的存在给我们提供了非常多可能性。在我们的方案中，涉及到的功能包括但不限于：
- 为用户创建的集群提供统一的日志分析、性能数据收集等功能；
- 为用户创建的k8s集群提供错误预测、告警等智能功能；
- 为用户集群提供统一的负载均衡策略；
- 为用户集群提供统一的集群升级及应用维护功能；
- 为用户集群提供备份执行机制，包括高可用应用的备份、尖峰负载的扩容等。

这些功能的设计及实现思考我们将在后续陆续探讨。

# 总结
LKS为基于k8s的应用开发及运维用户提供了一个自主、自如的k8s service功能。基于我们实现过程中的managed设计以及production设计，我们最大程度的减少了用户使用LKS的后顾之忧，从而可以大力促进容器系统的快速落地。