---
title: k8s container中docker commit提交镜像发现entrypoint变化
description: 
categories:
- k8s
tags:
- docker
---

最近发现一个很有意思的现象，在一个k8s已经运行起来的容器上执行docker commit保存镜像时我们可以发现，原来的容器CMD将被设置成新镜像的ENTRYPOINT。

# 现象
原有镜像的信息如下所示，docker inspect emma:vd查看镜像信息如下：


            "Cmd": [
                "bash"
            ],
            "Image": "",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,


在非k8s环境中通过docker run -it emma:vd /root/run.sh start启动了容器70b69df0ce69，并通过docker commit 70b69df0ce69 emma:vd保存修改之后的镜像，再次通过docker inspect emma:vd查看镜像信息如下：

            "Cmd": [
                "/root/run.sh",
                "start"
            ],
            "Image": "",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },

而当容器通过k8s pod的方式启动时，pod的command是["/root/run.sh", "start"]，此时我们通过docker exec登录到该容器并且执行docker commit提交镜像是发现镜像的cmd和entrypoint已经改变：

            "Cmd": null,
            "Image": "",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": [
                "/root/run.sh",
                "start"
            ],

通过docker inspect pod对应的容器发现，容器的cmd/entrypoint已经与对应的docker image不同。

那么，也就是说，当我们通过pod的command指定pod启动命令时，k8s其实是将命令指定为entrypoint而将cmd清空了。

# k8s docker command

我们看k8s types中container的定义发现：

	type Container struct {
		// Optional: The docker image's entrypoint is used if this is not provided; cannot be updated.
		// Variable references $(VAR_NAME) are expanded using the container's environment.  If a variable
		// cannot be resolved, the reference in the input string will be unchanged.  The $(VAR_NAME) syntax
		// can be escaped with a double $$, ie: $$(VAR_NAME).  Escaped references will never be expanded,
		// regardless of whether the variable exists or not.
		// +optional
		Command []string
		// Optional: The docker image's cmd is used if this is not provided; cannot be updated.
		// Variable references $(VAR_NAME) are expanded using the container's environment.  If a variable
		// cannot be resolved, the reference in the input string will be unchanged.  The $(VAR_NAME) syntax
		// can be escaped with a double $$, ie: $$(VAR_NAME).  Escaped references will never be expanded,
		// regardless of whether the variable exists or not.
		// +optional
		Args []string

Comand对应的的是image中的entrypoint，args对应的是image中的cmd。 docker manager中的操作也验证了这一点：
	
	func setEntrypointAndCommand(container *v1.Container, opts *kubecontainer.RunContainerOptions, dockerOpts dockertypes.ContainerCreateConfig) {
		command, args := kubecontainer.ExpandContainerCommandAndArgs(container, opts.Envs)
	
		dockerOpts.Config.Entrypoint = dockerstrslice.StrSlice(command)
		dockerOpts.Config.Cmd = dockerstrslice.StrSlice(args)
	}

OK，最终的原因是我们对k8s pod中的command项的认识有偏差。

# docker commit
默认情况下，当正在提交生成镜像时，容器将暂停直到提交完成。这减小了在提交的过程中数据损坏的可能性。如果不想暂停容器，可以设置–pause选项为false。

–change选项用来应用Dockerfile指令到将要创建的镜像，包括Dockerfile指令CMD、ENTRYPOINT、ENV、EXPOSE、WORKDIR等。