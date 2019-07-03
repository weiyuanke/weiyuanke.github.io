---
title: Kubernetes三种Client的使用示例
date: 2019-07-03 16:55:45
tags:
    - kubernetes
---

kubernetes的Client库——go-client中提供了如下三种类型的client

ClientSet：可以访问集群中所有的原生资源，如pods、deployment等，是最常用的一种

dynamicClient: 可以处理集群中所有的资源，包括crd（自定义资源），另外它的返回是一个map[string]interface{}类型；目前主要用在garbage collector和namespace controller中。

RestClient：前面两种client的基础， 更为底层一些。

相关的示例如下：

```
package main

import (
	"flag"
	"fmt"
	"k8s.io/api/core/v1"
	v12 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/runtime/serializer"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

var (
	//集群配置文件路径
	kubeconfigStr = flag.String("kubeconfig", "default value", "kubernetes config file")
)

func main()  {
	//解析参数
	flag.Parse()

	testClientSet()

	fmt.Println("\nrest")

	testRestClient()

	fmt.Println("\n.....")
	testDynamicClient()
}

func testRestClient() {
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfigStr)
	if err != nil {
		panic(err)
	}

    //原生接口都在/api下，扩展接口在/apis下
	config.APIPath = "/api"
	//pods资源相关的group为空
	config.GroupVersion = &schema.GroupVersion{
		Group:    "",
		Version: "v1",
	}
	//序列化方式，目前json和protocal buf
	config.ContentType = runtime.ContentTypeJSON
	config.NegotiatedSerializer = serializer.DirectCodecFactory{CodecFactory: scheme.Codecs}

	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err)
	}

	podList := v1.PodList{}
	//除了Do()方法之外，还有DoRaw()，返回原始的bytes； Do()会做一下类型的转化
	restClient.Get().Resource("pods").Namespace("").Do().Into(&podList)

	fmt.Println(podList)
}

func testClientSet() {
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfigStr)
	if err != nil {
		panic(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	//一行代码指定group、version、resource、以及动作
	podList, err := clientset.CoreV1().Pods("").List(v12.ListOptions{})

	fmt.Println(podList.Items)
}

func testDynamicClient() {
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfigStr)
	if err != nil {
		panic(err)
	}

	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	//指定group、version以及要访问的资源 
	testrcGVR := schema.GroupVersionResource{
		Group:    "",
		Version:  "v1",
		Resource: "pods",
	}

	unstr, err := dynamicClient.Resource(testrcGVR).List(v12.ListOptions{})

	fmt.Println(unstr)
}
```


