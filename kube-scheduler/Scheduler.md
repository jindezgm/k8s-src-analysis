<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-26 21:14:30
 * @Description: 
-->


# 前言

[调度算法](./ScheduleAlgorithm.md)只对调度周期从PreFilter到Score扩展点进行抽象和实现，那么调度算法又是如何与其他扩展点配合实现Pod的调度呢？这就是本文重点解析的内容，kube-scheduler定义了Scheduler类型，它实现了调度Pod的全流程。

本文引用源码为kubernetes的release-1.21分支。

# Scheduler

## Scheduler定义

Scheduler要想实现调度一个Pod的全流程，那么必须有[调度队列](./SchedulingQueue.md)、[调度缓存](./Cache.md)、[调度框架](./Framework.md)、[调度插件](./Plugin.md)、[调度算法](./ScheduleAlgorithm.md)等模块的支持。所以，Scheduler的成员变量势必会包含这些类型，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L62>

```go
// Scheduler监视未调度的Pod并尝试找到适合的Node，并将绑定信息写回到apiserver。
type Scheduler struct {
	// 调度缓存，用来缓存所有的Node状态，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Cache.md
	// 了解过调度算法的读者应该知道，调度缓存也会被调度算法使用，用来每次调度前更新快照
	SchedulerCache internalcache.Cache
    // 调度算法，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/ScheduleAlgorithm.md
	Algorithm core.ScheduleAlgorithm

	// NextPod()获取下一个需要调度的Pod(如果没有则阻塞当前协程)，是不是想到了调度队列(SchedulingQueue)了？
	// 那么问题来了，为什么不直接使用调度队列(下面有调度队列的成员变量)而是注入一个函数呢？笔者会在后续的章节中给出答案
	NextPod func() *framework.QueuedPodInfo

	// 当调度一个Pod出现错误的时候调用Error()函数，是Scheduler使用者注入的错误函数，相当于回调。
	Error func(*framework.QueuedPodInfo, error)

	// 关闭Scheduler的信号
	StopEverything <-chan struct{}

	// 调度队列，用来缓存等待调度的Pod，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/SchedulingQueue.md
	SchedulingQueue internalqueue.SchedulingQueue

	// 别看名字是Profile，其实是Framework，因为Profile和Framework是一对一的，Profile只是Framework的配置而已。
    // 详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Framework.md 
	Profiles profile.Map

	// apiserver的客户端，主要用来向apiserver执行写操作，比如写Pod的绑定信息
	client clientset.Interface
}
```

与笔者猜测Scheduler的定义基本差不多，没“料到”注入了NextPod()和Error()函数，因为这两个函数是使用者注入的，实现必然在Scheduler之外，笔者会在必要的章节对这两个注入函数加以注释。

## schedulerOptions

虽然Scheduler的成员变量包含了很多模块，但是不代表这些模块都是Scheduler构造的，按照惯例很多都是通过参数、选项传入进来的，schedulerOptions就是构造Scheduler的选项类型，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L91>

```go
// schedulerOptions定义了构造Scheduler的选项。
type schedulerOptions struct {
	// 调度算法源当前版本已经不推荐使用了，笔者在调度插件的文章中简单提到了。
	// 取而代之的是KubeSchedulerProfile(见下面的成员变量)，所以本文暂不做详细说明。
	schedulerAlgorithmSource schedulerapi.SchedulerAlgorithmSource
	// 这个是调度算法计算最多可行节点的比例阈值，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/ScheduleAlgorithm.md
	percentageOfNodesToScore int32
	// 这两个是调度队列的初始/最大退避时间，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/SchedulingQueue.md
	podInitialBackoffSeconds int64
	podMaxBackoffSeconds     int64
	// kube-scheduler的插件注册表分为InTree和OutTree两种，前一种就是在调度插件包内静态编译的，后一种是通过选项传入的。
    // frameworkOutOfTreeRegistry就是通过选项传入的插件工厂注册表，当前版本这个选项还是空的，没有用。
	// 但是scheduler-plugin这个兴趣组项目就是采用该选项实现插件的扩展: https://github.com/kubernetes-sigs/scheduler-plugins
	frameworkOutOfTreeRegistry frameworkruntime.Registry
	// 这个是调度框架(Framework)的配置，每个KubeSchedulerProfile对应一个调度框架，本文不会对该类型做详细解析，因为会有单独的文章专门解析kube-scheduler的配置。
	// 详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/KubeSchedulerConfiguration.md#KubeSchedulerProfile
	// 此处只需要知道一点：每个KubeSchedulerProfile配置了一个调度框架每个扩展点使用哪些调度插件以及调度插件的参数。
	profiles                   []schedulerapi.KubeSchedulerProfile
	// 调度扩展程序的配置，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Extender.md
	extenders                  []schedulerapi.Extender
	// 调度框架捕获器，因为这个选项对于理解Scheduler没有任何帮助，所以本文不做详细说明。
	// 详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Configurator.md#Configurator
	frameworkCapturer          FrameworkCapturer
	// 最大并行度，调度算法是多协程过滤、评分，其中最大协程数就是通过该选项配置的
	parallelism                int32
}
```

知道了Scheduler的定义，以及构造Scheduler的选项类型，接下来看看kube-scheduler是如何构造Scheduler的，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L190> 

```go
// New()是Scheduler的构造函数，其中clientset,informerFactory,recorderFactory是调用者传入的，所以构造函数不用创建。
// opts就是在默认的schedulerOptions叠加的选项，golang这种用法非常普遍，不需要多解释。
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	// 初始化stopEverything，如果调用者没有指定则永远不会停止(这种情况不可能，因为要优雅退出)。
	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

    // 在默认的schedulerOptions基础上应用所有的opts，其中defaultSchedulerOptions读者建议读者自己看一下
	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

	// 构造调度缓存，还记得第一个参数30秒是干什么的么？是绑定的超时阈值(TTL)，指定时间内没有确认假定调度的Pod将会被从Cache中移除
	schedulerCache := internalcache.New(30*time.Second, stopEverything)

	// 创建InTree的插件工厂注册表，并与OutTree的插件工厂注册表合并，形成最终的插件工厂注册表。
	// registry是一个map类型，键是插件名称，值是插件的工厂(构造函数)
	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	// 初始化Cache的快照
	snapshot := internalcache.NewEmptySnapshot()

	// 关于Configurator参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Configurator.md
	// 笔者简单剧透一下，Configurator用于根据配置构造Scheduler。
	configurator := &Configurator{
		client:                   client,
		recorderFactory:          recorderFactory,
		informerFactory:          informerFactory,
		schedulerCache:           schedulerCache,
		StopEverything:           stopEverything,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		podInitialBackoffSeconds: options.podInitialBackoffSeconds,
		podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
		registry:                 registry,
		nodeInfoSnapshot:         snapshot,
		extenders:                options.extenders,
		frameworkCapturer:        options.frameworkCapturer,
		parallellism:             options.parallelism,
	}

	metrics.Register()

	var sched *Scheduler
	// 根据算法源调用Configurator不同的接口构造Scheduler。
	// 因为算法源已经不推荐使用，所以可以认为是用configurator.createFromProvider()构造Scheduler。
	source := options.schedulerAlgorithmSource
	switch {
	// 基于Provider的算法源
	case source.Provider != nil:
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
	// 基于策略的算法源，本文忽略本段代码
	case source.Policy != nil:
		policy := &schedulerapi.Policy{}
		switch {
		case source.Policy.File != nil:
			if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
				return nil, err
			}
		case source.Policy.ConfigMap != nil:
			if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
				return nil, err
			}
		}
		configurator.extenders = policy.Extenders
		sc, err := configurator.createFromConfig(*policy)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler from policy: %v", err)
		}
		sched = sc
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
	// 设置Scheduler的Clientset和关闭信号
	sched.StopEverything = stopEverything
	sched.client = client

	// 注册事件处理函数，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/EventHandlers.md
	// 笔者简单剧透一下，就是通过SharedIndexInformer监视Pod、Node、Service等调度依赖的API对象，并根据事件类型执行相应的操作。
	// 其中新建的Pod放入调度队列、Pod绑定成功更新调度缓存都是事件处理函数做的。
	addAllEventHandlers(sched, informerFactory)
	return sched, nil
}
```

## scheduleOne

万事具备，只欠东风，现在可以开始解析Scheduler调度一个Pod的全流程实现，并且函数名字也非常应景，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L441>

```go
// scheduleOne()调度一个Pod。
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	// 获取下一个需要调度Pod，可以理解为从调用ScheduleingQueuePop()，为什么要注入一个函数呢？下面会有注入函数的解析。
	podInfo := sched.NextPod()
	// 调度队列关闭的时候返回空的Pod，说明收到了关闭的信号，所以直接退出就行了，不用再判断ctx
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
    // 根据Pod指定的调度器名字(Pod.Spec.SchedulerName)选择Framework。
	pod := podInfo.Pod
	fwk, err := sched.frameworkForPod(pod)
	if err != nil {
		klog.ErrorS(err, "Error occurred")
		return
	}
	// 是否需要忽略这个Pod，至于什么样的Pod会被忽略，后面有相关函数的注释。
	if sched.skipPodSchedule(fwk, pod) {
		return
	}

	klog.V(3).InfoS("Attempting to schedule pod", "pod", klog.KObj(pod))


	// 为调度Pod做准备，包括计时、创建CycleState以及调度上下文(schedulingCycleCtx)
	start := time.Now()
	state := framework.NewCycleState()
	state.SetRecordPluginMetrics(rand.Intn(100) < pluginMetricsSamplePercent)
	schedulingCycleCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	// 通过调度算法找到最优的Node，详情参看笔者关于调度算法的文章。
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, fwk, state, pod)
	if err != nil {
		// 调度算法可能因为任何Node都无法满足Pod资源需求返回失败，因此尝试进行抢占，并期望下一次尝试调度Pod时，由于抢占Node可以满足Pod的需求。
		// 也有一种可能存在，那就是另一个Pod被调度到被抢占的资源中，但这并没有什么损失，因为这说明这个Pod优先级更高。
		nominatedNode := ""
		// 判断调度算法返回的错误是否是因为资源无法满足造成的，如果是，那么就尝试抢占
		if fitError, ok := err.(*framework.FitError); ok {
			// 如果没有PostFilterPlugin(即抢占插件)，也就不用尝试抢占了
			if !fwk.HasPostFilterPlugins() {
				klog.V(3).InfoS("No PostFilter plugins are registered, so no preemption will be performed")
			} else {
				// 运行所有的PostFilterPlugin，尝试让Pod可在以未来调度周期中进行调度。
				// 为什么是未来的调度周期？道理很简单，需要等被强占Pod的退出。
				result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatusMap)
				if status.Code() == framework.Error {
					klog.ErrorS(nil, "Status after running PostFilter plugins for pod", klog.KObj(pod), "status", status)
				} else {
					klog.V(5).InfoS("Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", status)
				}
				// 如果抢占成功，则记录提名Node的名字。
				if status.IsSuccess() && result != nil {
					nominatedNode = result.NominatedNodeName
				}
			}
			// metrics相关的代码不做注释
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else if err == core.ErrNoNodesAvailable {
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else {
			klog.ErrorS(err, "Error selecting node for pod", "pod", klog.KObj(pod))
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		}
		// recordSchedulingFailure()用于统一实现调度Pod失败的处理，函数名字有record关键字，可以推测至少应该包含记录Pod调度失败的代码。
		// 也就是我们用kubectl describe pod xxx时，Events部分描述Pod因为什么原因不可调度的，所以参数有错误代码、不可调度原因等就很容易理解了。
		// 需要注意的是，即便抢占成功，Pod当前依然是不可调度状态，因为需要等待被强占的Pod退出，所以nominatedNode是否为空就可以判断是否抢占成功了。
		// 因为调度Pod失败的点非常多，后面有很多处都调用了recordSchedulingFailure()函数，笔者就不在重复注释了。
		// 下面有recordSchedulingFailure()函数注释，届时会揭开它神秘的面纱。
		sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
		return
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	// 深度拷贝PodInfo赋值给假定调度Pod，为什么深度拷贝Pod？因为assume()会设置Pod.Status.NodeName = scheduleResult.SuggestedHost
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod
	// assume()会调用Cache.AssumePod()假定调度Pod，assume()函数下面有注释，此处暂时认为Cache.AssumePod()就行了。
	// 需要再解释一下为什么要假定调度Pod，因为Scheduler不用等到绑定结果就可以调度下一个Pod，如果无法理解建议阅读笔者关于调度缓存的文章。
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// 什么情况会造成假定调度失败？根据Cache的源码可知Pod如果已经假定调度了，再次假定调度就会报错。
		// 那什么情况会重复假定调度Pod？根据Scheduler的事件处理函数源码可知，只要Pod未绑定，Add/Update事件都会将Pod放入调度队列。
		// 也就是在绑定周期事件内，如果Pod删除再添加亦或是更新，都有可能造成Pod重新调度并再次假定调度。
		// 还好，在注入的Error()函数中会检测Pod是否已经绑定，如果已经绑定则不会重新放入调度队列(否则，这将导致无限循环)。
		sched.recordSchedulingFailure(fwk, assumedPodInfo, err, SchedulerError, "")
		return
	}

	// 为Pod预留需要的全局资源，比如PV
	if sts := fwk.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// 即便预留资源失败了，还是调用一次恢复，可以清理一些状态，笔者认为理论上RunReservePluginsReserve()应该保证一定的事务性
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
        // 因为Cache中已经假定Pod调度了，此处就应该删除假定调度的Pod
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
		}
		// 见上面的注释
		sched.recordSchedulingFailure(fwk, assumedPodInfo, sts.AsError(), SchedulerError, "")
		return
	}

	// 判断Pod是否可以进入绑定阶段。
	runPermitStatus := fwk.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	// 有插件没有批准Pod并且不是等待，那只能是拒绝或者发生了内部错误
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		// 获取失败的原因
		var reason string
		if runPermitStatus.IsUnschedulable() {
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = v1.PodReasonUnschedulable
		} else {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = SchedulerError
		}
		// 从此处开始，一旦调度失败，都会做如下几个事情 ：
		// 1. 恢复预留的资源：fwk.RunReservePluginsUnreserve()；
		// 2. 删除假定调度的Pod：sched.SchedulerCache.ForgetPod()；
		// 3. 汇报调度失败：sched.recordSchedulingFailure()；
		// 所以后面相同的代码笔者就不在重复注释了，简单一句话：见上面的注释
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, runPermitStatus.AsError(), reason, "")
		return
	}

	// 进入绑定周期，创建一个协程异步绑定，因为绑定是一个相对比较耗时的操作，至少包含一次向apiserver写入绑定信息的操作。
	// 如果此时还不理解调度周期和绑定周期的读者，建议阅读笔者关于调度框架的文章。
	go func() {
		// 创建绑定周期的上下文，此时需要注意的是，现在已经处于另一个协程，Scheduler.scheduleOne已经开始调度下一个Pod了。
		// 从并行的角度讲，这属于时间并行(类似于流水线)。
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()
		metrics.SchedulerGoroutines.WithLabelValues(metrics.Binding).Inc()
		defer metrics.SchedulerGoroutines.WithLabelValues(metrics.Binding).Dec()

		// 等待Pod批准通过，如果有PermitPlugin返回等待，Pod就会被放入waitingPodsMap直到所有的PermitPlug批准通过。
		// 好在调度框架帮我们实现了这些功能，此处只需要调用一个接口就全部搞定了。
		waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
		if !waitOnPermitStatus.IsSuccess() {
			// 获取失败的原因
			var reason string
			if waitOnPermitStatus.IsUnschedulable() {
				metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = SchedulerError
			}
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, "")
			return
		}

		// 绑定预处理，详情参看笔者关于调度插件和调度框架的文章
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, "")
			return
		}

		// 执行绑定操作，所谓绑定，就是向apiserver写入Pod的子资源/bind，里面包含有选择的Node。
		// 单独封装bind()函数用意是什么呢？下面有bind()函数的注释，到时候就明白了。
		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.ErrorS(err, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, "")
		} else {
			if klog.V(2).Enabled() {
				klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
			}
			metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
			metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
			metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

			// 绑定后处理
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```

有没有感觉scheduleOne()就那么回事？就是按照扩展点的顺序逐一执行，如果出错就报错就可以了。此时如果感觉简单，那是因为对[调度队列](./SchedulingQueue.md)、[调度缓存](./Cache.md)、[调度框架](./Framework.md)、[调度插件](./Plugin.md)、[调度算法](./ScheduleAlgorithm.md)等模块已经有了充分的认识，如果没有这些知识的前期铺垫，相信小白一股脑扎进来肯定是满脸的懵逼。

接下来笔者对scheduleOne()调用的、Scheduler单独封装的一些函数进行解析，兑现在代码注释中的承诺。既然当前已经比较通透了，再通透点无妨。

### NextPod

笔者一直无法猜测注入NextPod的目的是什么，因为Scheduler有调度队列的成员变量，直接从调度队列队列中弹出不就行了么？毕竟调度队列实现了协程安全、阻塞等，还有什么考虑需要单独封装一个函数呢？ 源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/internal/queue/scheduling_queue.go#L845>

```go
// MakeNextPodFunc()返回一个函数，用于从给定的调度队列中获取下一个Pod
func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo {
	// 到此我也没什么好说的了，加了日志，这样所有需要弹出待调度Pod的地方就不用重复写日志了，这是非常有用的。
	// 千算万算没算到日志...
	return func() *framework.QueuedPodInfo {
		podInfo, err := queue.Pop()
		if err == nil {
			klog.V(4).InfoS("About to try and schedule pod", "pod", klog.KObj(podInfo.Pod))
			return podInfo
		}
		klog.ErrorS(err, "Error while retrieving next pod from scheduling queue")
		return nil
	}
}
```

### skipPodSchedule

在调度一个Pod时，第一件事情就是判断Pod是否需要忽略调度，那么有哪些情况需要忽略调度呢？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L634>

```go
// skipPodSchedule()判断Pod是否忽略调度。
func (sched *Scheduler) skipPodSchedule(fwk framework.Framework, pod *v1.Pod) bool {
	// 第一种情况，Pod被删除了，这个应该好理解，已经删除的Pod不需要要调度。
	// 那么问题来了，是不是可以在Pod更新事件中感知Pod.DeletionTimestamp的变化，然后从调度队列中删除呢？
	if pod.DeletionTimestamp != nil {
		fwk.EventRecorder().Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		klog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return true
	}

    // 第二种情况，Pod被更新但是已经假定被调度了，为什么会出现这种情况？
    // 这是深度拷贝的原因，scheduleOne()函数在调用假定调度的时候会深度拷贝Pod，然后设置Pod.Status.NodeName。
	// 在绑定周期，使用的Pod就是假定调度Pod，而不是调度队列中的Pod，此时如果更新了Pod，就会出现这种情况。
    // 因为还没有绑定完成，此时apiserver中Pod依然还是未调度状态，更新Pod势必会将Pod放入调度队列。
    // 此处不明白可以查看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/EventHandlers.md
    // 放入调度队列就会被再次调度，所以需要跳过它。需要注意的是，并不是所有的更新都会忽略调度。
	// 比如Pod的资源需求、标签、亲和性等调整了，势必会影响调度，而以前调度的结果应该被覆盖，所以不能够忽略。
	// skipPodUpdate()函数就是用来检测Pod的更新是否会影响调度，详情参看笔者关于事件处理的文章(上面EventHandlers.md的连接)。
	if sched.skipPodUpdate(pod) {
		return true
	}

	return false
}
```

### recordSchedulingFailure

前面笔者提到了recordSchedulingFailure()至少包含记录调度失败事件的功能，还应该有什么功能？有没有发现注入的Scheduler.Error()函数还没有用上？Pod从[调度队列](./SchedulingQueue.md)弹出Pod如果调度失败是不是应该放回[调度队列](./SchedulingQueue.md)？这部分实现笔者断定会放在recordSchedulingFailure()函数中，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L322> 

```go
// recordSchedulingFailure()用于调度Pod失败后的相关处理。
func (sched *Scheduler) recordSchedulingFailure(fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatedNode string) {
    // 这才是笔者注释recordSchedulingFailure()的关键，因为在scheduleOne()函数中通篇没有看到将Pod放回队列的过程.
    // 理论上只要出错，Pod应该放回到调度队列，否则这个Pod就丢失了。也就调度出错了才会放回队列，所以把Pod放回队列的功能放在了Error()函数中。
	// 这也是注入Error()函数的一个原因吧，下面有Error()的注释。
	sched.Error(podInfo, err)

    // 此处需要考虑两种情况：
    // 1. Pod已经提名Node，现在调度失败了，此时的nominatedNode == ""，AddNominatedPod()会恢复以前的提名
    // 2. Pod抢占调度成功，但是需要等待被强占的Pod退出，所以Pod提名了Node，此时的nominatedNode != ""
	if sched.SchedulingQueue != nil {
		sched.SchedulingQueue.AddNominatedPod(podInfo.Pod, nominatedNode)
	}

    // 记录Pod调度失败事件，也就是通过kubectl describe pod xxx时看到的Events
	pod := podInfo.Pod
	msg := truncateMessage(err.Error())
	fwk.EventRecorder().Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", msg)
	// 更新Pod的Status字段，因为调度失败是Pod的一种状态，包括提名的Node也是。
	if err := updatePod(sched.client, pod, &v1.PodCondition{
		Type:    v1.PodScheduled,
		Status:  v1.ConditionFalse,
		Reason:  reason,
		Message: err.Error(),
	}, nominatedNode); err != nil {
		klog.Errorf("Error updating pod %s/%s: %v", pod.Namespace, pod.Name, err)
	}
}
```

是时候揭开注入Scheduler.Error()函数的神秘面纱了，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/factory.go#L330>

```go
// MakeDefaultErrorFunc()构造一个函数来处理Pod调度错误。
// 至于是谁调用了这个函数，参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Configurator.md#create
func MakeDefaultErrorFunc(client clientset.Interface, podLister corelisters.PodLister, podQueue internalqueue.SchedulingQueue, schedulerCache internalcache.Cache) func(*framework.QueuedPodInfo, error) {
	return func(podInfo *framework.QueuedPodInfo, err error) {
		// 根据错误代码类型打印日志
		pod := podInfo.Pod
		if err == core.ErrNoNodesAvailable {
			klog.V(2).InfoS("Unable to schedule pod; no nodes are registered to the cluster; waiting", "pod", klog.KObj(pod))
		} else if fitError, ok := err.(*framework.FitError); ok {
			podInfo.UnschedulablePlugins = fitError.Diagnosis.UnschedulablePlugins
			klog.V(2).InfoS("Unable to schedule pod; no fit; waiting", "pod", klog.KObj(pod), "err", err)
		} else if apierrors.IsNotFound(err) {
			klog.V(2).InfoS("Unable to schedule pod, possibly due to node not found; waiting", "pod", klog.KObj(pod), "err", err)
			// 看看是不是因为未找到Node引起的错误
			if errStatus, ok := err.(apierrors.APIStatus); ok && errStatus.Status().Details.Kind == "node" {
				nodeName := errStatus.Status().Details.Name
				// 当找不到Node时，不会立即删除该Node。再次尝试通过apiservers获取该节点，如果仍然找不到该节点，则将其从调度缓存中删除。
				_, err := client.CoreV1().Nodes().Get(context.TODO(), nodeName, metav1.GetOptions{})
				if err != nil && apierrors.IsNotFound(err) {
					node := v1.Node{ObjectMeta: metav1.ObjectMeta{Name: nodeName}}
					if err := schedulerCache.RemoveNode(&node); err != nil {
						klog.V(4).InfoS("Node is not found; failed to remove it from the cache", "node", node.Name)
					}
				}
			}
		} else {
			klog.ErrorS(err, "Error scheduling pod; retrying", "pod", klog.KObj(pod))
		}

		// 参数pod是从调度队列中获取的，此时需要校验一下SharedIndexInformer的缓存中是否存在，如果不存在说明已经被删除了，也就不用再放回队列了。
		// 其实判断不存在并不是核心目的，只是顺带手的事，核心目的是缓存(非调度缓存)的Pod状态最新的，这样才能将最新的Pod放回到调度队列中。
		// 即便此时判断Pod存在，但是与此同时如果收到了Pod删除事件，那么很可能出现刚刚删除的Pod又被添加到调度队列中。
		cachedPod, err := podLister.Pods(pod.Namespace).Get(pod.Name)
		if err != nil {
			klog.InfoS("Pod doesn't exist in informer cache", "pod", klog.KObj(pod), "err", err)
			return
		}

		// 在Extender的情况下，Pod可能已成功绑定，但是其响应超时返回给Scheduler。这可能会导致实时版本带有Pod.Spec.NodeName，并且与内部排队版本不一致。
		if len(cachedPod.Spec.NodeName) != 0 {
			klog.InfoS("Pod has been assigned to node. Abort adding it back to queue.", "pod", klog.KObj(pod), "node", cachedPod.Spec.NodeName)
			return
		}

		// 备份缓存中的Pod，然后将其放回到调度队列中，当然是以不可调度的名义放回的。
		podInfo.PodInfo = framework.NewPodInfo(cachedPod.DeepCopy())
		if err := podQueue.AddUnschedulableIfNotPresent(podInfo, podQueue.SchedulingCycle()); err != nil {
			klog.ErrorS(err, "Error occurred")
		}
	}
}
```

现在应该明白为啥需要注入Error函数了，因为调度错误的处理并不是简单的放回到[调度队列](./SchedulingQueue.md)，期间还要根据Pod最新的状态再做决定。而最新的Pod状态存储在SharedIndexInformer的缓存中，而Scheduler没有SharedIndexInformer成员变量，注入函数的方式完美的解决了这个问题。

### assume

注入NextPod()是因为从[调度队列](./SchedulingQueue.md)中弹出Pod时写日志，难道单独封装assume()函数也是这个目的？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L330>

```go
// assume()是Scheduler假定Pod调度的处理函数
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
	// 通知Cache假定调度到'host'指定的Node上
	assumed.Spec.NodeName = host
	if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
		klog.ErrorS(err, "scheduler cache AssumePod failed")
		return err
	}
	// 仅仅通知Cache就完了么？如果Pod被提名了Node呢？因为Pod已经被(假定)调度了，相关的提名就要去掉，因为只有未调度的Pod才能被提名。
	// 调度队列实现了PodNominator，所以通过调度队列删除Pod的提名状态。这就可以理解单独封装assume()函数的目的了。
	if sched.SchedulingQueue != nil {
		sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
	}

	return nil
}
```

### bind

最后就是bind()函数，单独封装的原因比较简单，因为有两个模块可以实现绑定：BindPlugin和Extender，用一个函数统一实现绑定属于常规编程范畴。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/scheduler.go#L395>

```go
// 将Pod绑定到的给定Node，绑定的优先级是：（1）Extender和（2）BindPlugin。
func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (err error) {
	defer func() {
		// finishBinding()函数下面有注释。
		sched.finishBinding(fwk, assumed, targetNode, err)
	}()

	// 优先用Extender来绑定，下面有extendersBinding()的源码注释
	bound, err := sched.extendersBinding(assumed, targetNode)
	if bound {
		return err
	}
	// 如果所有Extender都没有绑定能力，则用BindPlugin
	bindStatus := fwk.RunBindPlugins(ctx, state, assumed, targetNode)
	if bindStatus.IsSuccess() {
		return nil
	}
	if bindStatus.Code() == framework.Error {
		return bindStatus.AsError()
	}
	return fmt.Errorf("bind status: %s, %v", bindStatus.Code().String(), bindStatus.Message())
}

// extendersBinding()调用Extender执行绑定，返回值为是否绑定成功和错误代码。
func (sched *Scheduler) extendersBinding(pod *v1.Pod, node string) (bool, error) {
	// 遍历所有的Extender.
	for _, extender := range sched.Algorithm.Extenders() {
		// 不是Extender有绑定能力(extender.IsBinder() == true)就可以绑定任何Pod，
        // 还有一个前提条件就是Pod的资源是Extender管理的才行，比如只申请CPU和内存的Pod用BindPlugin绑定它不香么？
		if !extender.IsBinder() || !extender.IsInterested(pod) {
			continue
		}
		// 调用Extender执行绑定。
		return true, extender.Bind(&v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name, UID: pod.UID},
			Target:     v1.ObjectReference{Kind: "Node", Name: node},
		})
	}
	return false, nil
}

// finishBinding()是绑定结束后的处理。
func (sched *Scheduler) finishBinding(fwk framework.Framework, assumed *v1.Pod, targetNode string, err error) {
	// 通知Cache绑定结束，详情参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Cache.md#FinishBinding
	if finErr := sched.SchedulerCache.FinishBinding(assumed); finErr != nil {
		klog.ErrorS(finErr, "scheduler cache FinishBinding failed")
	}
	// 如果绑定出现错误，则写日志
	if err != nil {
		klog.V(1).InfoS("Failed to bind pod", "pod", klog.KObj(assumed))
		return
	}

	// 记录绑定成功事件，那么问题来了，为什么绑定失败不记录事件？很简单，在recordSchedulingFailure()记录。
	fwk.EventRecorder().Eventf(assumed, nil, v1.EventTypeNormal, "Scheduled", "Binding", "Successfully assigned %v/%v to %v", assumed.Namespace, assumed.Name, targetNode)
}

```

# 总结

1. Scheduler从调度队列中弹出Pod，通过[调度算法](./ScheduleAlgorithm.md)选出最优的Node；
2. 如果无Node能够满足Pod资源，则通过PostPlugin实现抢占；
3. 如果选出了最优Node，则经过[ReservePlugin](./Plugin.md#ReservePlugin)和[PermitPlugin](./Plugin.md#PermitPlugin)后Pod进入绑定周期；
4. Scheduler创建独立的协程执行绑定操作，自己的协程则开始下一个Pod的调度周期；
5. 调度Pod的流程中，如果失败则需要恢复预留的资源、删除假定调度、送回到调度队列等一系列操作，具体要看在哪个阶段失败的；
6. BindPlugin和Extender都有绑定能力，那么优先使用Extender绑定，但还有一个前提条件，就是Pod的有些资源是由Extender管理才行。否则，该Pod申请的资源与Extender没有任何关系，由Extender执行绑定没有任何意义，反而效率更低了；
