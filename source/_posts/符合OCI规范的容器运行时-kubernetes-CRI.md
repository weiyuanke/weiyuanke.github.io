---
title: 符合OCI规范的容器运行时 & kubernetes CRI
date: 2019-11-08 12:37:52
tags:
---
参考[前一篇文章](https://blog.csdn.net/weiyuanke/article/details/90639505)

目前的docker已经不是之前的docker了，技术栈进行了分层。
docker cli -> dockerd -> containerd -> oci implementation

OCI规范，简单来说，包含容器规范和镜像规范，具体参考：
[link](https://github.com/opencontainers/runtime-spec)
由于引入了OCI，我们可以近乎透明无缝的替换 “docker run”命令 之下的具体实现。
根据自己业务场景的需求，可以选择高性能的容器运行时，也可以选择性能不那么高但安全性更好的运行时。

目前符合OCI规范的容器运行时有：
- runc， docker默认的运行时，与宿主机共享内核，利用cgroup做资源隔离，安全性不是很高，由于内核共享，性能最好
- runv，基于hypervisor的容器运行时，有自己单独的内核，安全性好很多
- kata，被蚂蚁收了，号称“容器的速度，虚拟机的安全”，貌似是基于runv做的
- gvisor，谷歌搞的，比runc安全，比VM性能要好。有一个用户空间的内核，会拦截应用程序的系统调用，目前没有实现所有的系统调用，因此不是所有的应用都可以运行

目前docker默认使用的容器运行时runc。

docker命令行有一个参数：runtime，可以制定底层的容器运行时，如

```
docker run --runtime=runc hello-world
```

runc官方有一个实例，介绍如何用runc运行一个容器

```
EXAMPLE:
  To run docker's hello-world container one needs to set the args parameter
in the spec to call hello. This can be done using the sed command or a text
editor. The following commands create a bundle for hello-world, change the
default args parameter in the spec from "sh" to "/hello", then run the hello
command in a new hello-world container named container1:

    mkdir hello
    cd hello
    docker pull hello-world
    docker export $(docker create hello-world) > hello-world.tar
    mkdir rootfs
    tar -C rootfs -xf hello-world.tar
    runc spec
    sed -i 's;"sh";"/hello";' config.json
    runc run container1
```

网上有一个runc 和 runv 运行示例的文章，参考: [链接](https://feisky.xyz/posts/2016-04-28-runc/)

说回到kubernetes，k8s抽象出了一个CRI接口。

kubelet 通过该接口与底层不同的容器运行时进行交互，从而实现解耦。
其中的关系如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191108120751489.png)
kubelet通过gRPC方式调用CRI的相关接口，CRI shim是该接口的一个实现。

kubelet有一个选项“--container-runtime”，
默认为docker，可以理解为非CRI模式。
设置为remote的时候，可以认为是启用了CRI模式，通过另外一个选项
--container-runtime-path-endpoint=<path> 来指定CRI服务的地址，一般是一个Linux 本地Socket。

kubelet会通过指定的CRI地址来进行容器的管理。



