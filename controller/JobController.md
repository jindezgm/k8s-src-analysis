<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-31 22:09:43
 * @Description: JobController源码解析
-->

# 前言

关于Job的介绍请参看[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)。笔者提取几个关键的的信息：

1. Job会创建一个或者多个Pods，如何指定创建几个Pod？
2. 直到指定数量的Pods成功终止，如何指定多少个Pod成功终止才算Job完成？
3. 在有些情形下，希望Job在经历若干次失败重试之后直接进入失败状态，如何指定重试次数？

以上问题在[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)都有答案，本文将从Job的API定义以及JobController源码的角度看看Kubernetes是如何实现Job的功能的。

本文引用源码为kubernetes的release-1.21分支。

# API(Job)

Job的API定义了Job的参数(配置、规格都行，反正意思到了就行)和Job的状态，前言中提到的几个问题都可以在API的定义找到答案。源码链接：<https://github.com/kubernetes/api/blob/release-1.21/batch/v1/types.go#L28>

```go
type Job struct {
    // 这两个没什么好说的了，所有的API对象都有
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	// Job的参数，可以理解为Job的常量，是Job的目标状态，详情参看JobSpec的注释。
	// 当然Job可能会更新，更新后的Job.Spec是不会变化的直到下一次Job更新为止。
	// 这段时间可能很短也可能很长，笔者将Job.Spec定义为常量就是相对于这段时间内的。
	Spec JobSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	// Job的状态，可以理解为Job的变量，是Job的当前状态，详情参看JobStatus的注释。
	// JobController就是计算Job.Spec与Job.Status的差异(diff)，然后执行一些操作后，这些操作执行成功后diff就会为0。
	// 当然执行的某些操作可能会失败，Job.Spec在执行这些操作的过程中也可能发生变化，JobController就是在无限的计算差异并修复差异。
	Status JobStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

// JobSpec是有用户指定的Job目标状态。
type JobSpec struct {
    // 指定Job在任何时候运行的Pod的最大期望数目，有时候实际运行的Pod数量可能会小于最大并行度。
    // 比如，当Job.Spec.Completions-Job.Status.Successful < Job.Spec.Parallelism，即当剩下要启动的Pod数量小于最大并行度时。
	// 这个变量回答了前言的问题1.
	Parallelism *int32 `json:"parallelism,omitempty" protobuf:"varint,1,opt,name=parallelism"`

    // Job完成需要Pod成功终止的数量，设置为nil表示任何Pod的成功都表示所有Pod的成功，并且最大并行度可以是任意正值.
	// 设置为1意味着并行度被限制为1，并且pod的成功表示Job已成功。这个变量回答了前言的问题2。
	Completions *int32 `json:"completions,omitempty" protobuf:"varint,2,opt,name=completions"`

    // 从Job.Status.StartTime开始持续时间(以秒为单位)超过ActiveDeadlineSeconds，Job则会失败，必须为正整数(如果需要设置)。
	ActiveDeadlineSeconds *int64 `json:"activeDeadlineSeconds,omitempty" protobuf:"varint,3,opt,name=activeDeadlineSeconds"`

    // 作业失败的重试次数，默认为6，这个变量回答了前言的问题3.
	BackoffLimit *int32 `json:"backoffLimit,omitempty" protobuf:"varint,7,opt,name=backoffLimit"`

    // Job匹配Pod的选择器，一般是系统为我们设置这个字段，没什么好解释的
	Selector *metav1.LabelSelector `json:"selector,omitempty" protobuf:"bytes,4,opt,name=selector"`

    // 是否人工生成Pod的标签以及选择器。除非你确定自己在做什么，否则请不要设置这个字段。
    // 设置为false或未设置时，系统会选择该Job唯一的标签，并将这些标签追加到Pod模板。如果为true，则用户负责选择唯一标签并指定选择器。
    // 标签不唯一可能会导致此Job和其他Job无法正常运行。
	ManualSelector *bool `json:"manualSelector,omitempty" protobuf:"varint,5,opt,name=manualSelector"`

    // Pod模板，本文不会展开介绍它，读者感兴趣可以自己了解一下。
	Template v1.PodTemplateSpec `json:"template" protobuf:"bytes,6,opt,name=template"`

	// 已完成Job（包括成功或失败）的生命周期：
    // 1. 如果设置了此字段，Job完成后会在*TTLSecondsAfterFinished秒后被自动删除;
    // 2. 如果未设置此字段，则不会自动删除Job;
    // 3. 如果此字段设置为零，则Job完成后立即删除;
    TTLSecondsAfterFinished *int32 `json:"ttlSecondsAfterFinished,omitempty" protobuf:"varint,8,opt,name=ttlSecondsAfterFinished"`

	// Job的完成模式，包含'NonIndexed'(默认)和'Indexed'两种。
	// 'NonIndexed'意思是Completions上来数量的Pod成功完成，Job就完成了；
	// 'Indexed'意思是Job的所有Pod都关联了[0, Completions-1]范围内的索引，在Pod.Annotation["batch.kubernetes.io/job-completion-index"]可以看到。
	// 当所索引的的Pod都完成后Job才算是完成，当*CompletionMode == "Indexed"时，Completions必须设置有效值。
	// IndexedJob还是处于alpha阶段，需要使能IndexedJob的feature才能使用。
	CompletionMode *CompletionMode `json:"completionMode,omitempty" protobuf:"bytes,9,opt,name=completionMode,casttype=CompletionMode"`

	// 表示Job是否挂起，挂起的Job不能够创建Pod。如果Job已经创建了Pod，比如Suspend有false变成true，已创建的Pod会被删除。
	Suspend *bool `json:"suspend,omitempty" protobuf:"varint,10,opt,name=suspend"`
}

// JobStatus是由JobController填写的Job当前状态
type JobStatus struct {
    // Job完成情况，比如成功或者失败，详情参看JobCondition的代码注释。
	Conditions []JobCondition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
    // Job开始时间
	StartTime *metav1.Time `json:"startTime,omitempty" protobuf:"bytes,2,opt,name=startTime"`
    // Job完成的时间。
	CompletionTime *metav1.Time `json:"completionTime,omitempty" protobuf:"bytes,3,opt,name=completionTime"`
    // 活跃运行的Pod数。
	Active int32 `json:"active,omitempty" protobuf:"varint,4,opt,name=active"`
    // Pod.Status.Phase = Succeeded的Pod数量，当JobStatus.Succeeded >= *JobSpec.Completions时Job完成
	Succeeded int32 `json:"succeeded,omitempty" protobuf:"varint,5,opt,name=succeeded"`
    // Pod.Status.Phase = Failed的Pod数量，当JobStatus.Failed >= *JobSpec.BackoffLimit时Job失败
	Failed int32 `json:"failed,omitempty" protobuf:"varint,6,opt,name=failed"`
}

// JobCondition作业完成情况
type JobCondition struct {
    // JobCondition类型， "Complete"或者"Failed"，前者是成功完成，后者是失败未完成。
	Type JobConditionType `json:"type" protobuf:"bytes,1,opt,name=type,casttype=JobConditionType"`
    // JobCondition状态，"True"、"False"、"Unknown"之一，当前只用到了"True"
	Status v1.ConditionStatus `json:"status" protobuf:"bytes,2,opt,name=status,casttype=k8s.io/api/core/v1.ConditionStatus"`
    // 这两个变量在JobController虽然被赋值，但是并未引用
	LastProbeTime metav1.Time `json:"lastProbeTime,omitempty" protobuf:"bytes,3,opt,name=lastProbeTime"`
	LastTransitionTime metav1.Time `json:"lastTransitionTime,omitempty" protobuf:"bytes,4,opt,name=lastTransitionTime"`
    // 这两个变量用于记录Job失败的原因，如果是成功完成都为""
	Reason string `json:"reason,omitempty" protobuf:"bytes,5,opt,name=reason"`
	Message string `json:"message,omitempty" protobuf:"bytes,6,opt,name=message"`
}
```

# JobController

## Controller

了解了Job的API的定义，是不是对JobController的职责多少有一点认识了？因为Kubernetes中Job对象非常多，JobController负责将所有的Job当前状态(JobStatus)调整到Job的目标状态(JobSpec)，这个调整就是对Job的"控制"，也就是Controller的来源吧(存储笔者自己猜测)。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L66>

```go
type Controller struct {
    // Controller需要更新Job的状态(JobStatus)，肯定需要Clientset。
	// 至于创建、删除Pod是通过PodControl实现的，而PodControl也是需要Clientset的。
	kubeClient clientset.Interface
    // 用于操作(比如创建、删除)Job的Pod，参看https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/PodControl.md
	podControl controller.PodControlInterface

	// updateHandler用于将Job更新到apiserver，定义成函数是方便单元测试时函数注入，否则就真的需要一个apiserver了。
	updateHandler func(job *batch.Job) error
	// syncHandler用于同步处理一个Job对象，定义成函数也是方便单元测试时函数注入，实际它指向了Controller的成员函数。
	syncHandler   func(jobKey string) (bool, error)
    // 这两个变量指向了SharedIndexInformer.HasSynced()函数，分别用于判断Cache是否已经同步了apiserver的Pod和Job。
    // SharedIndexInformer参看https://github.com/jindezgm/k8s-src-analysis/blob/master/client-go/tools/cache/SharedIndexInformer.md
	podStoreSynced cache.InformerSynced
	jobStoreSynced cache.InformerSynced

	// 用于Controller记录每个Job期望创建和删除的Pod数量。
    // ControllerExpectationsInterface参看https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/ControllerExpectations.md
	expectations controller.ControllerExpectationsInterface

	// 这两个变量指向了SharedIndexInformer.Indexer()，分别用于从Cache中获取Job和Pod，注意是从Cache中而不是apiserver。
    // SharedIndexInformer参看https://github.com/jindezgm/k8s-src-analysis/blob/master/client-go/tools/cache/SharedIndexInformer.md
	jobLister batchv1listers.JobLister
	podStore corelisters.PodLister

	// Job的队列，所有待处理的Job都会放在队列中，Controller从队列中取出Job对象然后处理它
	queue workqueue.RateLimitingInterface
    // 用于记录每个Job的事件，比如创建Pod失败。
	recorder record.EventRecorder
}
```

从Controller的定义可以看出: 


1. Controller通过JobInformer和PodInformer感知Job的状态变化（Pod是Job的子对象，是Job状态的一部分）；
2. 利用PodControl创建/更新/删除Pod达到Job的目标状态，利用Clientset更新Job的状态；
3. 利用Queue存储Job等待Controller处理；

## NewController

NewController是JobController的构造函数，用于kube-controller-manager创建JobController对象，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L97>

```go
unc NewController(podInformer coreinformers.PodInformer, jobInformer batchinformers.JobInformer, kubeClient clientset.Interface) *Controller {
	// 事件相关的代码不是本文重点内容，后面笔者不会对相关的代码注释
    eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
    
    // Metrics相关的代码笔者也不做注释
	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		ratelimiter.RegisterMetricAndTrackRateLimiterUsage("job_controller", kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}
    
    // 创建Controller对象，相关的代码比较简单.
	jm := &Controller{
		// Clientset是kube-controller-manager创建，供所有的Controller共享使用的
		kubeClient: kubeClient,
        // 创建PodControl
		podControl: controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "job-controller"}),
		},
        // 创建ControllerExpectations
		expectations: controller.NewControllerExpectations(),
        // 创建队列
		queue:        workqueue.NewNamedRateLimitingQueue(workqueue.NewItemExponentialFailureRateLimiter(DefaultJobBackOff, MaxJobBackOff), "job"),
		recorder:     eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "job-controller"}),
	}

    // 注册Job事件的处理函数，jobInformer是通过SharedInformerFactory创建的，所有的Controller共享SharedIndexInformer。
	jobInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        // 处理新的Job对象事件，enqueueController会将obj(Job)放入队列中，enqueueController下面有代码注释
		AddFunc: func(obj interface{}) {
			jm.enqueueController(obj, true)
		},
        // 处理Job对象更新事件，updateJob()函数下面有注释
		UpdateFunc: jm.updateJob,
        // 处理Job对象删除事件，删除Pod也放入队列？这说明Controller只会根据Job的当前状态做事。
		// 放入队列无非是通知Controller该Job状态发生了变化，看看是不是能做点什么？
		DeleteFunc: func(obj interface{}) {
			jm.enqueueController(obj, true)
		},
	})
    // 这个没什么好解释的，为什么用函数指针，直接调用不就可以了么？道理很简单，为了方便单元测试注入函数。否则单元测试还需要JobInformer，这就比较麻烦了。
	jm.jobLister = jobInformer.Lister()
	jm.jobStoreSynced = jobInformer.Informer().HasSynced

    // 下面的代码和上面一样，无非对变成了Pod，后面有Pod各种事件处理函数的注释
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    jm.addPod,
		UpdateFunc: jm.updatePod,
		DeleteFunc: jm.deletePod,
	})
	jm.podStore = podInformer.Lister()
	jm.podStoreSynced = podInformer.Informer().HasSynced

    // 别问为什么，问就是为了函数注入，都是单元测试的锅。
	jm.updateHandler = jm.updateJobStatus
	jm.syncHandler = jm.syncJob

	return jm
}
```

## Run

Controller的构造函数并没有看到任何创建协程的代码，Controller不应该有个协程从队列中取出Job处理么，我们写某个对象的构造函数习惯最后一行写go xxx.run()。Controller也有一个类似的函数(Run)，但不是在构造函数中调用的，而是由kube-controller-manager调用，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L146>

```go
// Run()函数创建多个worker协程处理Job对象。
func (jm *Controller) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer jm.queue.ShutDown()

	klog.Infof("Starting job controller")
	defer klog.Infof("Shutting down job controller")

	// 等待Job对象从apiserver同步到Cache中
	if !cache.WaitForNamedCacheSync("job", stopCh, jm.podStoreSynced, jm.jobStoreSynced) {
		return
	}

	// workers是kube-controller-manager的一个配置项，也就是有多个协程并发处理队列中的Job对象。
	// 此处需要思考一个问题：为什么需要多个协程？有的同学肯定会说这很正常哈，每个协程处理一个Job对象，效率更高。
	// 笔者不以为然，高效率是相对而言的，举个例子，一个协程处理100个Job对象只需要100ms，假设用10个协程需要10ms。
	// 虽然效率提升了10倍，但是对于使用者是无感的，也就是100ms和10ms的感觉是一样的。
	// 采用多协程并发处理是因为处理单个Job可能会比较耗时，比如更新Job状态需要一次apiserver的写入。
	for i := 0; i < workers; i++ {
		go wait.Until(jm.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

Controller除了自己创建的worker协程处理Job对象外，还有SharedIndexInformer的协程驱动Controller处理Job和Pod的事件。在解析worker协程之前，我们先看看各个事件的处理函数都做了些什么。

## enqueueController

在Controller的构造函数中可以看到，Job的Add/Delete事件处理函数直接调用了enqueueController()函数，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L361>

```go 
// enqueueController()将obj放入Controller.queue中，immediate为true表示立刻放入队列。
// 需要注意: Controller注册的Add/Delete处理函数immediate都是true，即立刻放入队列
func (jm *Controller) enqueueController(obj interface{}, immediate bool) {
    // controller.KeyFunc会用obj的NS/NAME生成唯一键
	key, err := controller.KeyFunc(obj)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for object %+v: %v", obj, err))
		return
	}

    // 计算对象的退避时间，如果immediate == true则退避时间为0，否则按照对象错误重试次数计算退避时间。
	// getBackoff()的感兴趣的源码读者可以自己看下，此处就不展开了。
	backoff := time.Duration(0)
	if !immediate {
		backoff = getBackoff(jm.queue, key)
	}

    // 将对象的唯一键放入队列中，此处需要注意的是: 
	// 1. 队列中缓存的是Job的唯一键(NS/Name)；
	// 2. AddAfter()并不会累加对象重试次数，不了解这一点的读者可以阅读笔者关于队列的文章。
	jm.queue.AddAfter(key, backoff)
}
```

### updateJob

updateJob是Job的Update事件处理函数，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L329>

```go
func (jm *Controller) updateJob(old, cur interface{}) {
	oldJob := old.(*batch.Job)
	curJob := cur.(*batch.Job)

	// 获取Job的唯一键，NS/NAME
	key, err := controller.KeyFunc(curJob)
	if err != nil {
		return
	}
    // 将更新的Job放入队列中，所以可以的到结论，只要Job有任何事件都会将Job放入队列中，而且是立刻放入，唤醒worker来处理它。
	jm.enqueueController(curJob, true)
	// Job.Status.StartTime是worker协程第一次处理Job的时候设置的。
	// 与此同时，worker协程还会根据Job的Deadline(如果设置了)告知队列延迟添加(AddAfter)该Job来触发Job处理超时错误。
	// Job.Status.StartTime != nil说明Job已经被worker协程向队列延迟添加了，此时Job更新Deadline可能也发生了变化，所以需要处理一下。
	if curJob.Status.StartTime != nil {
        // 如果最新Job没有设置Deadline，则返回？为什么？因为只有设置了Deadline才需要向队列延迟添加。
		// 可能存在一种情况，curJob.Spec.ActiveDeadlineSeconds == nil && oldJob.Spec.ActiveDeadlineSeconds != nil。
		// 也就是说该Job已经向队列延迟添加，现在没有Deadline，为什么不告知队列取消延迟添加呢？
		// 其实这并没有什么问题，因为向队列添加(无论是否延迟)都是唤醒为了worker处理Job，所以多唤醒一次并不是问题。
		curADS := curJob.Spec.ActiveDeadlineSeconds
		if curADS == nil {
			return
		}
        // 此处再判断Deadline是否发生了变化(以前没有现在有，或者Deadline不同)，如果有变化就需要告知队列延迟Deadline秒后添加Job，来通知worker协程终止Job。
        // 为什么Deadline发生变化需要向队列延迟添加？因为老的Job如果设置了Deadline，那么worker协程处理Job的时候会向队列延迟添加。
        // 这里需要思考一个问题: 如果Deadline真的修改了，就会有2次延迟放入队列的操作，也无非是多唤醒一次worker处理Job。
		// 无论是提前唤醒(新的Deadline更长)还是延迟唤醒(新的Deadline更短)都不会有问题，因为worker每次都会根据Job最新的状态处理，后面就会看到了。
		oldADS := oldJob.Spec.ActiveDeadlineSeconds
		if oldADS == nil || *oldADS != *curADS {
            // 根据Deadline计算延迟时间
			now := metav1.Now()
			start := curJob.Status.StartTime.Time
			passed := now.Time.Sub(start)
			total := time.Duration(*curADS) * time.Second
			// 延迟放入队列
			jm.queue.AddAfter(key, total-passed)
			klog.V(4).Infof("job ActiveDeadlineSeconds updated, will rsync after %d seconds", total-passed)
		}
	}
}
```

## add/update/deletePod

add/update/deletePod()Controller处理Pod事件的函数，Controller为什么要处理Pod事件？因为Controller需要根据JobSpec创建/删除/更新Pod，而Pod事件是Controller确认Pod创建/删除/更新成功的方法。有读者肯定会问，Clientset的接口不是明确的返回了创建/删除/更新的错误代码了么，为什么还要通过Pod事件确认。这就需要提到Cache与apiserver一致性的问题了，Clientset的接口返回成功但是状态此时还没有同步到Cache，此时通过Cache获取的Pod和Job对象都不是最新的，自然不会处理正常。Controller需要等到相应的状态同步到Cache中然后再通知Controller处理，这就是Controller需要处理Pod事件的原因。

源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L204>

```go
// Controller处理创建Pod的事件。
func (jm *Controller) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)
	// 什么情况？addPod()居然要调用deletePod()？
	if pod.DeletionTimestamp != nil {
		// 当Controller重启时，已经删除的Pod可能会以Add事件通知Controller，此时的Pod.DeletionTimestamp不为空
		jm.deletePod(pod)
		return
	}

	// Pod是否有ControllerRef？即Pod是否有一个Onwer是Controller。
	// 关于ControllerRef参看https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/ControllerRefManager.md
	// GetControllerOf()就是返回Pod第一个Controller的Owner，感兴趣的读者可以自己看一下。
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		// resolveControllerRef()根据controllerRef解析出Job对象，下面有resolveControllerRef()函数注释。
		job := jm.resolveControllerRef(pod.Namespace, controllerRef)
		// job == nil说明Pod的Owner不是Job，亦或是Job的UID与controllerRef.UID不一致
		if job == nil {
			// 不属于Job的Pod不用做任何处理。
			return
		}
		// 获取Job的唯一键NS/Name
		jobKey, err := controller.KeyFunc(job)
		if err != nil {
			return
		}
		// Controller期望创建的Pod数量-1，此处需要理解ControllerExpectations，它是用来管理Controller创建和删除Pod的期望值。
		// 关于ControllerExpectations参看https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/ControllerExpectations.md
		jm.expectations.CreationObserved(jobKey)
		// 将Job再放入队列，为什么？如果此时读者还是简单的认为队列只是用来管理待处理的Job就大错特错了，此时建议把队列想象成sync.Cond。
		// 将Job放入队列就相当于调用了sync.Cond.Signal()或者sync.Cond.Broadcast()来通知worker协程来处理这个Job。
		// 至于worker协程需要做什么，就是根据JobSpec与JobStatus的差异执行相应的操作，后面会有详细的解析。
		// 当前属于Job的Pod创建成功了，说明JobStatus.Active发生了变化，就应该唤醒worker协程来处理这个Job。
		jm.enqueueController(job, true)
		return
	}

	// 如果Pod没有Controller的Owner，那么他就是一个孤儿，此时通过Pod的标签看看哪些Job预支匹配。
	for _, job := range jm.getPodJobs(pod) {
		// 将Job放入队列，唤醒worker协程处理这个Job，看看Job能够接纳这个Pod。
		jm.enqueueController(job, true)
	}
}

// resolveControllerRef()解析ControllerRef，返回引用的Job对象。
func (jm *Controller) resolveControllerRef(namespace string, controllerRef *metav1.OwnerReference) *batch.Job {
	// Owner是不是Job？
	if controllerRef.Kind != controllerKind.Kind {
		return nil
	}
	// 根据引用的名字获取Job
	job, err := jm.jobLister.Jobs(namespace).Get(controllerRef.Name)
	if err != nil {
		return nil
	}
	// 引用的ID与Job.UID不同，说明Pod的Owner是其他的Job，可能同名的Job被删除后又创建
	if job.UID != controllerRef.UID {
		return nil
	}
	return job
}

// Controller处理Pod更新事件
func (jm *Controller) updatePod(old, cur interface{}) {
	curPod := cur.(*v1.Pod)
	oldPod := old.(*v1.Pod)
	// 判断Pod是否真的更新，因为周期性的Resync会触发Update事件。
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		return
	}
	// 判断Pod是否已经删除，这个和addPod()相同
	if curPod.DeletionTimestamp != nil {
		jm.deletePod(curPod)
		return
	}

	// Pod更新对于Controller来说可以知道Pod是否完成，其中包括是失败。
	// Pod失败Job不会立刻放入队列，需要根据失败的次数计算退避时间延迟放入，所以此处用来判断是否需要立刻将Pod放入队列。
	immediate := curPod.Status.Phase != v1.PodFailed

	// 判断Job的Controller是否发生了变化
	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	// ControllerRef发生了变化，并且老的ControllerRef不为空
	if controllerRefChanged && oldControllerRef != nil {
		// 通知worker协程处理老的Job(如果有的话)，因为Pod已经不属于老的Job了
		if job := jm.resolveControllerRef(oldPod.Namespace, oldControllerRef); job != nil {
			jm.enqueueController(job, immediate)
		}
	}

	// 如果有ControllerRef，通知通知worker协程处理Job，因为Job有更新
	if curControllerRef != nil {
		job := jm.resolveControllerRef(curPod.Namespace, curControllerRef)
		if job == nil {
			return
		}
		jm.enqueueController(job, immediate)
		return
	}

	// Pod没有ControllerRef，那就是个孤儿，只有Pod的标签发生变化或者ControllerRef有变化的时候，看看哪些Job可以接纳这个Pod。
	// 为什么标签发生变化需要处理孤儿？因为上一个标签的Pod已经在addPod()或者updatePod()函数中执行完跟下面一样的代码。
	// ControllerRef发生变化说明以前有Owner，现在没有了，那么就要看看还有哪些Job可以接纳这个Pod
	labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
	if labelChanged || controllerRefChanged {
		for _, job := range jm.getPodJobs(curPod) {
			jm.enqueueController(job, immediate)
		}
	}
}

// Controller处理Pod删除事件
func (jm *Controller) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// 这部分不是本文的重点内容，笔者暂时不对这部分的逻辑做注释，会有专门的文档解析这部分逻辑。
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %+v", obj))
			return
		}
	}

	// 获取Pod的ControlerRef
	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		return
	}
	// 获取Pod引用的Job
	job := jm.resolveControllerRef(pod.Namespace, controllerRef)
	if job == nil {
		return
	}
	// 获取Job唯一键NS/Name
	jobKey, err := controller.KeyFunc(job)
	if err != nil {
		return
	}
	// 期望删除的Pod-1
	jm.expectations.DeletionObserved(jobKey)
	// 唤醒worker协程处理Job的原因有两个:
	// 1. Pod被删除，JobStatus.Active可能会变化；
	// 2. Job的期望可能已经达成；
	jm.enqueueController(job, true)
}

```

## worker

千呼万唤始出来，终于到了worker协程函数了，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L384>

```go
func (jm *Controller) worker() {
	// 从队列中取出一个Job然后处理，processNextWorkItem()下面有注释
	for jm.processNextWorkItem() {
	}
}

func (jm *Controller) processNextWorkItem() bool {
	// 取出Job，需要注意的是队列中存储的是Job的唯一键NS/Name，而不是Job对象
	key, quit := jm.queue.Get()
	if quit {
		return false
	}
	// 函数退出需要告知队列该Job已经处理完毕了，此处需要注意一点，Done()函数不是简单的将Job从队列中删除，不了解的读者可以参看笔者关于队列的文章。
	defer jm.queue.Done(key)

	// 同步处理Job，也就是说syncHandler()函数对Job是一次同步操作，可能比较耗时(比如向apiserver更新JobStatus、创建Pod)。
	// 所以Controller采用协程池的方式并发同步处理Job。
	forget, err := jm.syncHandler(key.(string))
	if err == nil {
		// forget代表什么？这和队列的原理有关，因为Controller创建的队列是一个限速队列，队列根据Job错误尝试次数以2的指数计算退避时间。
		// forget就是告诉队列忘记错误尝试次数，从0开始。详情参看笔者关于队列的文章。
		if forget {
			jm.queue.Forget(key)
		}
		return true
	}

	// 处理出错，需要根据错误尝试次数计算退避时间再延迟放入队列，需要注意的是AddRateLimited()会累加Job错误尝试次数。
	utilruntime.HandleError(fmt.Errorf("Error syncing job: %v", err))
	jm.queue.AddRateLimited(key)

	return true
}
```

## syncJob

Controller的worker协程函数的重点在syncJob()，笔者带大家看看同步处理Job都处理了啥？同步的意义是啥？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L443>

```go
func (jm *Controller) syncJob(key string) (bool, error) {
	// 获取当前时间，用于在日志中记录同步处理一个Job对象的时间
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing job %q (%v)", key, time.Since(startTime))
	}()

	// 获取Job的NS和Name
	ns, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return false, err
	}
	if len(ns) == 0 || len(name) == 0 {
		return false, fmt.Errorf("invalid job key %q: either namespace or name is missing", key)
	}
	// 根据Job的唯一key获取Job对象，Job对象是从Cache中获取的，队列只存储Job的唯一键，这样做有助于提升处理Job的效率。
	// 因为放入队列中只是为了唤醒worker协程处理这个Job，而只是为了处理当时那一刻的Job，就好像调用sync.Cond.Signal()多次和一次效果是一样的。
	// 相同的Job，worker协程被第一次放入队列唤醒，但是获取的Job的状态可能是第三次被放入队列时的状态。
	// 此时只要一次处理完成就可以把3次放入队列需要做的事情全部做完，而后面两次只是空转。
	sharedJob, err := jm.jobLister.Jobs(ns).Get(name)
	if err != nil {
		if apierrors.IsNotFound(err) {
			klog.V(4).Infof("Job has been deleted: %v", key)
			jm.expectations.DeleteExpectations(key)
			return true, nil
		}
		return false, err
	}
	// 深度拷贝Job，这样在函数中对Job的修改就不会影响到Cache中的Job
	job := *sharedJob.DeepCopy()

	// 如果Job已经完成了，就没什么好处理的了。如何判断Job完成了？IsJobFinished()下面有注释。
	// 这里也可以证明笔者的观点，就是syncJob()每次都根据Cache中Job的最新状态做处理，与具体是有什么事件触发无关。
	// 例如，一个Job的Add事件也需要判断Job是否完成，对于worker协程过来说Job在任何状态的处理流程都是一样的。
	if IsJobFinished(&job) {
		return true, nil
	}

	// 如果没有开启IndexedJob的feature，则不处理该类型的Job
	if !utilfeature.DefaultFeatureGate.Enabled(features.IndexedJob) && isIndexedJob(&job) {
		jm.recorder.Event(&job, v1.EventTypeWarning, "IndexedJobDisabled", "Skipped Indexed Job sync because feature is disabled.")
		return false, nil
	}
	// 不处理未知CompletionMode的Job
	if job.Spec.CompletionMode != nil && *job.Spec.CompletionMode != batch.NonIndexedCompletion && *job.Spec.CompletionMode != batch.IndexedCompletion {
		jm.recorder.Event(&job, v1.EventTypeWarning, "UnknownCompletionMode", "Skipped Job sync because completion mode is unknown")
		return false, nil
	}

	// 检测Job的期望是否达成，有没有发现jobNeedsSync在这里赋值，但是在很远的地方引用，为什么不在引用的时候再检测是否达成预期？
	// 这里面有一个时序的问题，下面要统计活跃(active)的Pod数量，如果在统计的过程中有Pod的Add事件，影响了Job的期望。
	// 问题就在于如果这个Pod没有被统计到活跃的Pod数量中就麻烦了，接下来的处理会因为期望达成但是active的Pod数量不够而再创建Pod。
	// 把校验Job的期望达成放在前面就不会有这问题，即便active多了一些还没有在期望值中减去的Pod，因为期望未达成也不会执行创建/删除Pod的操作。
	// 下一次处理该Job的时候可能就没有这个偏差了。
	jobNeedsSync := jm.expectations.SatisfiedExpectations(key)

	// 获取Job的所有Pod，需要注意的是getPodsForJob()还会接纳孤儿Pod，下面有函数注释。
	pods, err := jm.getPodsForJob(&job)
	if err != nil {
		return false, err
	}
	// 统计活跃的(active)、成功终止(succeeded)和失败终止(failed)的Pod
	activePods := controller.FilterActivePods(pods)
	active := int32(len(activePods))
	succeeded, failed := getStatus(&job, pods)
	// Job是第一次被处理，设置Job的开始时间并延迟Deadline(如果有的话)放入队列()，前提是Job没有被挂起。
	if job.Status.StartTime == nil && !jobSuspended(&job) {
		now := metav1.Now()
		job.Status.StartTime = &now
		if job.Spec.ActiveDeadlineSeconds != nil {
			klog.V(4).Infof("Job %s has ActiveDeadlineSeconds will sync after %d seconds",
				key, *job.Spec.ActiveDeadlineSeconds)
			jm.queue.AddAfter(key, time.Duration(*job.Spec.ActiveDeadlineSeconds)*time.Second)
		}
	}

	var manageJobErr error
	jobFailed := false
	var failureReason string
	var failureMessage string

	// 比上一次syncJob()有失败的Pod？
	jobHaveNewFailure := failed > job.Status.Failed
	// new failures happen when status does not reflect the failures and active
	// is different than parallelism, otherwise the previous controller loop
	// failed updating status so even if we pick up failure it is not a new one
	exceedsBackoffLimit := jobHaveNewFailure && (active != *job.Spec.Parallelism) &&
		(failed > *job.Spec.BackoffLimit)

	if exceedsBackoffLimit || pastBackoffLimitOnFailure(&job, pods) {
		// Job错误重试次数超过了阈值，作业失败
		jobFailed = true
		failureReason = "BackoffLimitExceeded"
		failureMessage = "Job has reached the specified backoff limit"
	} else if pastActiveDeadline(&job) {
		// Job执行时间超过了Deadline，作业失败
		jobFailed = true
		failureReason = "DeadlineExceeded"
		failureMessage = "Job was active longer than specified deadline"
	}

	// 
	var succeededIndexes string
	if isIndexedJob(&job) {
		succeededIndexes, succeeded = calculateSucceededIndexes(pods)
	}
	jobConditionsChanged := false
	manageJobCalled := false
	if jobFailed {
		// Job失败，删除所有活跃的Pod
		_, manageJobErr = jm.deleteJobPods(&job, "", activePods)

		// 更新Job的状态，在JobStatus.Conditions追加一个失败的Condition
		failed += active
		active = 0
		job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobFailed, v1.ConditionTrue, failureReason, failureMessage))
		jobConditionsChanged = true
		jm.recorder.Event(&job, v1.EventTypeWarning, failureReason, failureMessage)
	} else {
		if jobNeedsSync && job.DeletionTimestamp == nil {
			// Job的期望达成并且Job未被删除，就可以根据当前Pod状态，继续创建或者删除Pod，manageJob()专门负责这个事情，后面有注释。
			// 需要注意，新建的期望默认值是0，是期望已达成状态。
			active, manageJobErr = jm.manageJob(&job, activePods, succeeded, pods)
			manageJobCalled = true
		}
		// 根据JobSpec.Completions判断Job是否完成。
		completions := succeeded
		complete := false
		if job.Spec.Completions == nil {
			// JobSpec.Completions为空，只要有任何Pod成功都代表Job成功。
			if succeeded > 0 && active == 0 {
				complete = true
			}
		} else {
			// 成功完成的Pod数量大于等于指定的值，Job也达到了完成的状态。
			if completions >= *job.Spec.Completions {
				complete = true
				if active > 0 {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManyActivePods", "Too many active pods running after completion count reached")
				}
				if completions > *job.Spec.Completions {
					jm.recorder.Event(&job, v1.EventTypeWarning, "TooManySucceededPods", "Too many succeeded pods running after completion count reached")
				}
			}
		}
		if complete {
			// Job完成，更新状态，在JobStatus.Conditions追加完成的Condition。W
			job.Status.Conditions = append(job.Status.Conditions, newCondition(batch.JobComplete, v1.ConditionTrue, "", ""))
			jobConditionsChanged = true
			now := metav1.Now()
			job.Status.CompletionTime = &now
			jm.recorder.Event(&job, v1.EventTypeNormal, "Completed", "Job completed")
		} else if utilfeature.DefaultFeatureGate.Enabled(features.SuspendJob) && manageJobCalled {
			// Update the conditions / emit events only if manageJob was called in
			// this syncJob. Otherwise wait for the right syncJob call to make
			// updates.
			if job.Spec.Suspend != nil && *job.Spec.Suspend {
				// Job can be in the suspended state only if it is NOT completed.
				var isUpdated bool
				job.Status.Conditions, isUpdated = ensureJobConditionStatus(job.Status.Conditions, batch.JobSuspended, v1.ConditionTrue, "JobSuspended", "Job suspended")
				if isUpdated {
					jobConditionsChanged = true
					jm.recorder.Event(&job, v1.EventTypeNormal, "Suspended", "Job suspended")
				}
			} else {
				// Job not suspended.
				var isUpdated bool
				job.Status.Conditions, isUpdated = ensureJobConditionStatus(job.Status.Conditions, batch.JobSuspended, v1.ConditionFalse, "JobResumed", "Job resumed")
				if isUpdated {
					jobConditionsChanged = true
					jm.recorder.Event(&job, v1.EventTypeNormal, "Resumed", "Job resumed")
					// Resumed jobs will always reset StartTime to current time. This is
					// done because the ActiveDeadlineSeconds timer shouldn't go off
					// whilst the Job is still suspended and resetting StartTime is
					// consistent with resuming a Job created in the suspended state.
					// (ActiveDeadlineSeconds is interpreted as the number of seconds a
					// Job is continuously active.)
					now := metav1.Now()
					job.Status.StartTime = &now
				}
			}
		}
	}

	forget := false
	// 如果成功完成的Pod数量比上一次多，说明相比上一次处理Job有新的成功完成的Pod，那么就应该将Job的退避时间清零。
	// 这么做的目标是当parallelism > 1时，改善有些Pod失败有些Pod成功的情况。
	if job.Status.Succeeded < succeeded {
		forget = true
	}

	// 相比上一次处理Job，状态没有任何变化则不需要更新Job对象
	if job.Status.Active != active || job.Status.Succeeded != succeeded || job.Status.Failed != failed || jobConditionsChanged {
		job.Status.Active = active
		job.Status.Succeeded = succeeded
		job.Status.Failed = failed
		if isIndexedJob(&job) {
			job.Status.CompletedIndexes = succeededIndexes
		}

		// 更新Job，此处的更新是向apiserver发送更新请求。
		if err := jm.updateHandler(&job); err != nil {
			return forget, err
		}

		// 如果有新的失败的Pod，需要返回错误，这样Job对象会退避一段时间后再处理。
		if jobHaveNewFailure && !IsJobFinished(&job) {
			// returning an error will re-enqueue Job after the backoff period
			return forget, fmt.Errorf("failed pod(s) detected for job key %q", key)
		}

		forget = true
	}

	return forget, manageJobErr
}

// IsJobFinished()判断Job是否完成
func IsJobFinished(j *batch.Job) bool {
	for _, c := range j.Status.Conditions {
		// 只要有任何Condition是Complete或者Failed，说明Job已经失败了。
		if (c.Type == batch.JobComplete || c.Type == batch.JobFailed) && c.Status == v1.ConditionTrue {
			return true
		}
	}
	return false
}

// getPodsForJob()获取Job的Pods
func (jm *Controller) getPodsForJob(j *batch.Job) ([]*v1.Pod, error) {
	// 获取Job的标签选择器
	selector, err := metav1.LabelSelectorAsSelector(j.Spec.Selector)
	if err != nil {
		return nil, fmt.Errorf("couldn't convert Job selector: %v", err)
	}
	// 列举所有的Pod，有些Pod可能无法匹配Job的标签选择器，但是可能他们有指向该Job的ControllerRef
	pods, err := jm.podStore.Pods(j.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}
	// If any adoptions are attempted, we should first recheck for deletion
	// with an uncached quorum read sometime after listing Pods (see #42639).
	// 在https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/ControllerRefManager.md 中一直没有解释canAdopt。
	// 此处可以看看JobController的是怎么实现的，很简单，就是再次校验一下Job是否被删除，
	canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
		fresh, err := jm.kubeClient.BatchV1().Jobs(j.Namespace).Get(context.TODO(), j.Name, metav1.GetOptions{})
		if err != nil {
			return nil, err
		}
		if fresh.UID != j.UID {
			return nil, fmt.Errorf("original Job %v/%v is gone: got uid %v, wanted %v", j.Namespace, j.Name, fresh.UID, j.UID)
		}
		return fresh, nil
	})
	// 获取尝试获取Job的拥有权，其实绝大部分情况Controller都有这些Pod的拥有权，这么做就是为了解决一些孤儿Pod
	cm := controller.NewPodControllerRefManager(jm.podControl, j, selector, controllerKind, canAdoptFunc)
	return cm.ClaimPods(pods)
}

```

## manageJob

manageJob()是Controller创建/删除Pod的函数，即计算JobSpec与JobStatus的差异，然后根据差异创建/删除差异数量的Pod。根据上面的代码可知，manageJob()函数不是每次syncJob()的时候都调动，需要满足Job期望达成这个必要条件。所谓期望达成就是上次调用manageJob()的所有请求(异步创建/删除Pod的请求)都已完成，这样的实现方法我们平时的变成中用sync.WaitGroup实现。因为manageJob可能会创建/删除多个Pod，Controller确认Pod创建/删除是通过相应的事件，worker协程不能等待这些事件发生后在继续执行，所以用ControllerExpectations实现了Controller的期望。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/job/job_controller.go#L747>

```go
func (jm *Controller) manageJob(job *batch.Job, activePods []*v1.Pod, succeeded int32, allPods []*v1.Pod) (int32, error) {
	// 获取活跃的Pod数量和最大并行度
	active := int32(len(activePods))
	parallelism := *job.Spec.Parallelism
	// 获取Job的唯一键NS/Name
	jobKey, err := controller.KeyFunc(job)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for job %#v: %v", job, err))
		return 0, nil
	}

	// Job是否被挂起？
	if jobSuspended(job) {
		// 挂起的Job应该删除所有活跃的Pod
		klog.V(4).InfoS("Deleting all active pods in suspended job", "job", klog.KObj(job), "active", active)
		// 获得需要删除的Pod，一般情况下所有的活跃的Pod都需要删除，但是对于IndexedJob来说删除也是有顺序要求的。
		// activePodsForRemoval()函数感兴趣的读者可以自己看一下源码。
		podsToDelete := activePodsForRemoval(job, activePods, int(active))
		// 设置期望删除的Pod数量
		jm.expectations.ExpectDeletions(jobKey, len(podsToDelete))
		// 将需要删除的Pod删除。
		removed, err := jm.deleteJobPods(job, jobKey, podsToDelete)
		active -= removed
		return active, err
	}

	// 计算最少要删除的Pod数量，如果活跃的Pod数量小于等于最大并行度，那么就不用删除任何Pod。
	// 这就是在计算PodStatus.Active与PodSpec.Parallelism的差异，看看是不是多创建了Pod。
	rmAtLeast := active - parallelism
	if rmAtLeast < 0 {
		rmAtLeast = 0
	}
	// 获取需要删除的Pod，前面提到了activePodsForRemoval()可以针对IndexedJob处理。
	// 其实这个函数还可以做一个有意思的事情，就是对Pod进行排序，然后找到删除成本最低的Pod。
	// 排序的机制是: not-ready < ready, unscheduled < scheduled, and pending < running. 
	// 举一个最简单的例子，删除一个运行中的Pod肯定是比删除一个未调度的Pod成本高。
	// 所以activePodsForRemoval()可以在activePods中找出rmAtLeast个删除成本最低的Pod。
	podsToDelete := activePodsForRemoval(job, activePods, int(rmAtLeast))
	if len(podsToDelete) > 0 {
		// 设置删除Pod的期望值
		jm.expectations.ExpectDeletions(jobKey, len(podsToDelete))
		klog.V(4).InfoS("Too many pods running for job", "job", klog.KObj(job), "deleted", rmAtLeast, "target", parallelism)
		// 删除指定的Pod
		removed, err := jm.deleteJobPods(job, jobKey, podsToDelete)
		active -= removed
		if err != nil {
			return active, err
		}
	}

	// 如果活跃的Pod数量没有达到最大并行度，那么就要考虑创建Pod了，很好理解吧？
	if active < parallelism {
		wantActive := int32(0)
		if job.Spec.Completions == nil {
			// 在前面介绍Job的API的时候提到了，如果JobSpec.Completions == nil，代表可以创建最大并行度数量的Pod，但是只要有任何Pod成功终止都表示Job成功完成。
			// 此处判断成功终止的Pod数量，如果不为0就不用在创建Pod了，因为Job已经完成了，如果为0Pod的数量应该是最大并行度。
			if succeeded > 0 {
				wantActive = active
			} else {
				wantActive = parallelism
			}
		} else {
			// 根据已成功终止的Pod计算还剩下多少个Pod，然后与最大并行度取最小值就是期望活跃的Pod数量
			wantActive = *job.Spec.Completions - succeeded
			if wantActive > parallelism {
				wantActive = parallelism
			}
		}
		// 计算需要创建的Pod数量
		diff := wantActive - active
		if diff < 0 {
			// 需要创建的Pod数量反而是负数？这种情况笔者还没想到什么情况会发生，只当个异常情况处理吧，也就是不创建Pod了。
			utilruntime.HandleError(fmt.Errorf("More active than wanted: job %q, want %d, have %d", jobKey, wantActive, active))
			diff = 0
		}
		// 不需要创建Pod直接返回，相当于空转
		if diff == 0 {
			return active, nil
		}

		// 设置创建Pod的期望值
		jm.expectations.ExpectCreations(jobKey, int(diff))
		errCh := make(chan error, diff)
		klog.V(4).Infof("Too few pods running job %q, need %d, creating %d", jobKey, wantActive, diff)

		// 因为可能会创建多个Pod，需要多个协程并发创建，所以需要sync.WaitGroup来等待所有创建协程退出。
		wait := sync.WaitGroup{}

		var indexesToAdd []int
		if job.Spec.Completions != nil && isIndexedJob(job) {
			indexesToAdd = firstPendingIndexes(allPods, int(diff), int(*job.Spec.Completions))
			diff = int32(len(indexesToAdd))
		}
		// 先把即将创建的Pod数量加到活跃Pod数量中，后面创建失败的再减回来就好了
		active += diff

		podTemplate := job.Spec.Template.DeepCopy()
		if isIndexedJob(job) {
			addCompletionIndexEnvVariables(podTemplate)
		}

		// 分批创建Pod，第一批是1个，第二批是2个，第三批是4个，以此类推。为什么要这么做呢？这样可以避免启动大量因为相同错误而失败的Pod。
		// 举个栗子：比如一个低配额的项目无法创建大量的Pod，采用分批创建的方式一旦失败，后续批次就不用再向apiserver发送“垃圾请求”了。
		// 同时，也可以防止因为错误产生的各种“垃圾事件”。
		for batchSize := int32(integer.IntMin(int(diff), controller.SlowStartInitialBatchSize)); diff > 0; batchSize = integer.Int32Min(2*batchSize, diff) {
			// 记录当前有多少个错误在chan中，如果是第一次肯定是0。errorCount申请的容量是diff个，可以容纳创建所有Pod失败。
			errorCount := len(errCh)
			// wait用于等待当前批次所有Pod创建完成
			wait.Add(int(batchSize))
			for i := int32(0); i < batchSize; i++ {
				completionIndex := unknownCompletionIndex
				if indexesToAdd != nil {
					completionIndex = indexesToAdd[0]
					indexesToAdd = indexesToAdd[1:]
				}
				go func() {
					template := podTemplate
					if completionIndex != unknownCompletionIndex {
						template = podTemplate.DeepCopy()
						addCompletionIndexAnnotation(template, completionIndex)
					}
					defer wait.Done()
					// 利用PodControl创建Pod
					err := jm.podControl.CreatePodsWithControllerRef(job.Namespace, template, job, metav1.NewControllerRef(job, controllerKind))
					if err != nil {
						// 如果是NS被删除了，可以忽略这个错误，因为后续的所有创建都是错误的
						if apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
							return
						}
					}
					if err != nil {
						defer utilruntime.HandleError(err)
						// 期望创建的Pod数量减1，因为SharedIndexInformer不会捕获到该Pod创建事件。
						klog.V(2).Infof("Failed creation, decrementing expectations for job %q/%q", job.Namespace, job.Name)
						jm.expectations.CreationObserved(jobKey)
						// 此处加锁的原因是当前是一个独立的协程，存在并发访问的问题。为什么不考虑用原子呢？
						atomic.AddInt32(&active, -1)
						errCh <- err
					}
				}()
			}
			// 等待所有创建Pod的协程退出。
			wait.Wait()
			// errorCount是创建当前批次Pod错误上数量，如果当前批次Pod创建完成后errCh内的错误数量比以前多了，说明有Pod创建失败了。
			// 那么剩余的Pod就不要创建了，这就是skippedPods用来记录忽略Pod的原因。感觉判断len(errCh) > 0就行了，因为循环就此结束，errorCount肯定是0.
			skippedPods := diff - batchSize
			if errorCount < len(errCh) && skippedPods > 0 {
				klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for job %q/%q", skippedPods, job.Namespace, job.Name)
				// 获取Pod数量减去或略Pod的上诉量。
				active -= skippedPods
				// 同时期望创建Pod的数量也要减去忽略Pod的数量。
				for i := int32(0); i < skippedPods; i++ {
					jm.expectations.CreationObserved(jobKey)
				}
				// 一旦发现有错误就退出循环了，忽略的Pod以后再创建。那么我们想想以后是什么时候？
				// 很简单，就是当前期望创建的Pod创建完成，通过SharedIndexInformer通知Controller.
				break
			}
			// 需要创建Pod数量减去当前批次的数量
			diff -= batchSize
		}

		// 有任何错误都要返回错误
		if err := errorFromChannel(errCh); err != nil {
			return active, err
		}
	}

	return active, nil
}
```

# 总结

1. Controller无法为每个Job创建一个sync.Cond然后创建多个worker协程Wait这些sync.Cond，所以就用队列实现，将Job放入队列等同于worker协程处理它；
2. Jod的Add事件可以通知Controller处理一个新的Job；
3. Job的Delete事件可以通知Controller处理一个已删除的Job；
4. Job的Update事件基本等同于Add事件，但是需要对Deadline额外处理一下，因为worker协程只会处理新Job的Deadline;
5. Pod的Add事件对应于JobStatus.Active以及期望创建Pod数量的更新，所以需要唤醒worker协程处理该Job，如果Pod是孤儿，需要唤醒worker协程处理标签匹配的Job，看看是否可以接纳这些Pod；
6. Pod的Update事件包含: 1)ControllerRef发生了变化，处理是去拥有权的老的Job(如果有的话)，处理获得拥有权的新Job(如果有的话); 2)ControllerRef没有发生变化，处理拥有Pod的Job(如果有的话)，因为可能Pod已终止；3)孤儿Pod标签更新或者ControllerRef从有到无，所有标签匹配的Job看看是否可以接纳这个Pod；
7. Pod的Delete事件对应于JobStatus.Active以及期望删除Pod数量的更新，所以需要唤醒worker协程处理该Job
