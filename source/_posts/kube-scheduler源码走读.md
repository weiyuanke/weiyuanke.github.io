---
title: kube-scheduler源码走读
date: 2019-07-15 12:20:36
tags:
    - kubernetes
---

kube-scheduler是k8s中相对比较简单的一个服务，它监听api server获取新建的Pod，从众多的Node中选择一个合适的，来运行该Pod。

选择的过程分两个阶段：预选阶段 & 优选阶段

- 预选阶段：根据Pod创建的要求，筛选出所有符合要求的Node，通过该阶段的Node理论上都可以运行目标Pod
- 优选阶段：给上一步筛选出来的Node打分，选择一个分数最高的Node

本文简单的跟进一下kube-scheduler执行的整个流程。

入口代码：

```
#cmd/kube-scheduler/app/server.go:62
#同样基于cobra包开发，
	cmd := &cobra.Command{
		Use: "kube-scheduler",
		Long: `The Kubernetes scheduler is a policy-rich, topology-aware,
workload-specific function that significantly impacts availability, performance,
and capacity. The scheduler needs to take into account individual and collective
resource requirements, quality of service requirements, hardware/software/policy
constraints, affinity and anti-affinity specifications, data locality, inter-workload
interference, deadlines, and so on. Workload-specific requirements will be exposed
through the API as necessary.`,
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, args, opts); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}


// runCommand runs the scheduler.
func runCommand(cmd *cobra.Command, args []string, opts *options.Options) error {
	//构建调度所需的配置文件：主要包括kubeclient、eventclient、
	cc, err := opts.Config()

	stopCh := make(chan struct{})

	//根据当前的feature gates对调度的算法做一些裁剪
	// Apply algorithms based on feature gates.
	// TODO: make configurable?
	algorithmprovider.ApplyFeatureGates()

	//启动调度服务
	return Run(cc, stopCh)
}
```

继续看Run函数

```
// Run executes the scheduler based on the given configuration. It only return on error or when stopCh is closed.
func Run(cc schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}) error {
	// Create the scheduler.
	//构造一个scheduler，（Scheduler），
	//构造调度的预选策略列表、优选策略列表、为各个informer关联handler处理函数
	sched, err := scheduler.New(cc.Client, ...,

	// Start all informers.
	//启动各个infomer，监听相关的变化
	go cc.PodInformer.Informer().Run(stopCh)
	cc.InformerFactory.Start(stopCh)

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(stopCh)

	// Prepare a reusable runCommand function.
	run := func(ctx context.Context) {
		sched.Run()
		<-ctx.Done()
	}

	ctx, cancel := context.WithCancel(context.TODO()) // TODO once Run() accepts a context, it should be used here
	defer cancel()

	go func() {
		select {
		case <-stopCh:
			cancel()
		case <-ctx.Done():
		}
	}()


	// Leader election is disabled, so runCommand inline until done.
	run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

这里边重要的函数就两个，

一个是scheduler.New（） 构建了一个Scheduler对象，关联了各个informer的动作；

一个是run(ctx)，启动调度服务，run(ctx)最终会调用函数：scheduleOne；

先看下scheduleOne函数

```
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
	//从待调度队列中拿到一个需要调度的Pod
	pod := sched.config.NextPod()
	// pod could be nil when schedulerQueue is closed
	if pod == nil {
		return
	}

	//采用调度算法选择一个合适的Node来运行该Pod
	scheduleResult, err := sched.schedule(pod)

	assumedPod := pod.DeepCopy()
	
	#根据调度结果scheduleResult， 将pod绑定到对应的Node上
}
```




