---
title: tcp clientside grpc server solution
description: 
categories:
- k8s
tags:
- grpc
---
LKS is a kubernetes service runs on Openstack which you can know more from the previous article [lks k8s service](https://larryck.github.io/k8s/2018/12/12/lks-k8s-service/). In LKS, there is a grpc server runs on each nodes of user-created k8s cluster, which runs commands from Kube0. Accoring to the design of LKS, the connection between k8s cluster nodes and Kube0 can only be created from the nodes side, which means the grpcs server is also the client side of a tcp connection. 

# yamux

**yamux**  is a multiplexing library for Golang which relies on an underlying connection to provide reliability and ordering, such as TCP. yamux has also provide a mechanism to implement a grpc server from the client side of tcp connection.

Here is an example:

	conn, err := net.DialTimeout("tcp", *argConHost + ":" + *argConPort, time.Second * 5)
	if err != nil {
		log.Fatalf("error dialing: %s", err)
	}

	srvConn, err := yamux.Server(conn, yamux.DefaultConfig())
	if err != nil {
		log.Fatalf("couldn't create yamux server: %s", err)
	}

	// create a server instance
	s := api.Server{}

	// create a gRPC server object
	grpcServer := grpc.NewServer()

	// attach the Ping service to the server
	api.RegisterLksAdminServer(grpcServer, &s)

	// start the gRPC erver
	log.Println("launching gRPC server over TCP connection...")
	if err := grpcServer.Serve(srvConn); err != nil {
		log.Fatalf("failed to serve: %s", err)
	}

---

添加好友我们可以聊更多相关话题[二维码](https://upload-images.jianshu.io/upload_images/7924740-a905d0f137971f94.jpeg)