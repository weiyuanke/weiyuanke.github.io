---
title: PodWorkers
date: 2019-11-20 13:59:36
tags:
---

PodWorkers是最终“同步”pod状态的一块逻辑，
“同步”在这里的含义是：确保kubelet所在节点的Pod状态和etcd中的状态一致，该增加的增加，该删除的删除，该更新的更新。

“同步”动作的触发有几个方式：
（1）通过文件、apiserver、http方式监听到的变化
（2）定时器触发，例如每隔10s

PodWorkers的入口是UpdatePod

```
type podWorkers struct {
    // key是pod的ID，value是一个chan，会有一个gorouting监听该chann
    // 死循环一般的处理该Pod上的所有变动
	podUpdates map[types.UID]chan UpdatePodOptions
	
	// 记录每个pod对应的gorouting当前的工作状态
	isWorking map[types.UID]bool
	
	// 每个Pod对应的gorouting处理完相关逻辑之后，会把Pod的ID塞入
	// workQueue。定时器会触发kubelet，kubelet会从workQueue里
	// 获取需要同步的Pod，然后调用PodWorkers的UpdatePod方法，
	// 触发新一轮的同步
	workQueue queue.WorkQueue
}
```

简单的示意图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191120135846524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXl1YW5rZQ==,size_16,color_FFFFFF,t_70)
