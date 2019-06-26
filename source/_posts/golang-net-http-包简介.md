---
title: golang net/http 包简介
date: 2019-06-25 23:28:54
tags: 
	- golang
---

Kubernetes的Api Server中用到了golang的http包，故而做下简单的了解。

``` golang
package main

import (
	"fmt"
	"io"
	"net/http"
)

func main() {
	http.HandleFunc("/", helloworld)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Println("ListenAndServe: ", err)
	}
}

func helloworld(w http.ResponseWriter, request *http.Request) {
	result := "HelloWorld\n"
	result = result + "RequestURI: " + request.RequestURI

	io.WriteString(w, result)
}

```

以上代码开启了一个简单的web服务，应用启动后访问如下地址：

```
http://127.0.0.1:9090/?we=12
```
会给出如下的输出

```
HelloWorld
RequestURI: /?we=12
```

```
//第一个参数是监听的地址和端口
//第二个参数是一个Handler，实现了ServeHTTP(ResponseWriter, *Request)方法，
//客户端的请求会交由这个方法进行处理。
//如果传nil的话，会采用http包默认的handler，DefaultServeMux来进行处理。
//DefaultServeMux可以认为是一个路由管理器，
//它的ServeHTTP方法实现 会根据客户端请求的路径来将请求路由到不同的HandlerFunc上去。
http.ListenAndServe(":9090", nil)

//为DefaultServeMux注册一个HandleFunc
http.HandleFunc("/", helloworld)

具体的代码：
// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

除了采用默认的路由分发器DefaultServeMux，也可以构建自己的Mux，如下：

```
package main

import (
	"io"
	"net/http"
)

func main() {
	//新建一个路由管理器，
	mux := http.NewServeMux()
	//绑定路径和处理函数
	mux.HandleFunc("/", helloworld)

	server := &http.Server{
		Addr:":9090",
		Handler:mux,
	}

	server.ListenAndServe()
}

func helloworld(w http.ResponseWriter, request *http.Request) {
	result := "HelloWorld\n"
	result = result + "RequestURI: " + request.RequestURI

	io.WriteString(w, result)
}
```