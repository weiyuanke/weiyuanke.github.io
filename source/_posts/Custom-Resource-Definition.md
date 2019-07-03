---
title: Custom Resource Definition简介
date: 2019-07-01 14:33:30
tags:
	- kubernetes
---

1、什么是自定义资源（Custom Resource）？

```
自定义资源是对kubernetes api的扩展，用户通过配置文件（Custom Resource Definition）
定义 资源 后，就可以通过kubernetes的API来操作此类资源。

仅仅通过API对资源进行操作其实意义不大，一般会再配合一个自定义的控制器，来实现一个声明式的
API，在自定义的控制器中监听该资源相关的事件，并执行对应的业务逻辑。
```

2、自定义资源 -> 自定义控制器的整个流程

```
API server（包含有自定义资源） -> Lister&Watcher -> Reflector -> Delta Queue ->
Informer -> 存入Indexer并同时触发相关的Handler -> work queue -> 自定义控制器
```