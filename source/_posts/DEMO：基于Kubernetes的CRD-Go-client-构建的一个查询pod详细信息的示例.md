---
title: DEMO：基于Kubernetes的CRD & Go-client 构建的一个查询pod详细信息的示例
date: 2019-07-02 11:49:32
tags:
	- kubernetes
---


本文是一个示例程序，展示一下如何基于Kubernetes的CRD（Custom Resource Defination）& Go-client构建一个查询Pod详细信息的服务。

该服务运行起来后，可以通过标准的kubernetes api接口，基于CRD来查询某个Pod的详细信息。

代码的地址在：

```
https://github.com/weiyuanke/PodInfoLookup
```

首先我们基于配置文件创建一个我们自定义的资源——Testcr

```
#PodInfoLookup/yaml/testcrd.yaml

#创建了一个名为Testcr的自定义资源类型
#group：stable.example.com
#version：v1
#resource：testcrs
$kubectl apply -f yaml/testcrd.yaml
```

本文假定本地已经有一个kubernetes集群在运行了，如果没有的话，可以基于minikube安装一个。

```
#拉取最新代码：
go get github.com/weiyuanke/PodInfoLookup

#启动服务
go run main.go --kubeconfig=/Users/yuankewei/.kube/config
```

在另外一个终端创建一个pod

```
kubectl apply -f yaml/pod.yaml 

#服务会监听到pod的创建，同时将该pod的相关信息写入Testcr资源中，
add:  default/nginx

current list:
&{map[kind:Testcr metadata:map[generation:1 name:29b36e2c6835dded8a115aee874d1ddc resourceVersion:113140 selfLink:/apis/stable.example.com/v1/testcrs/29b36e2c6835dded8a115aee874d1ddc uid:a2440f67-9ca4-11e9-9f23-080027f1737d creationTimestamp:2019-07-02T08:37:32Z] spec:map[podip:nginx podkey:a242bc2d-9ca4-11e9-9f23-080027f1737d podname:29b36e2c6835dded8a115aee874d1ddc poduid:] apiVersion:stable.example.com/v1]}

#同理，删除pod，会导致Testrc中对应的资源被删除
kubectl delete -f yaml/pod.yaml 

delete:  default/nginx

current list:

```

