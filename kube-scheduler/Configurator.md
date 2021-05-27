<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-27 22:31:43
 * @Description: 
-->

# 前言

[调度器配置](./KubeSchedulerConfiguration.md)中解析了kube-scheduler的各种配置，但是自始至终都没有看到如何根据配置构造[调度器](./Scheduler.md)。虽然[调度器](./Scheduler.md)文章中提到了构造函数，但是其核心实现是Configurator，本文将解析Configurator构造调度器的过程。虽然意义远没有[调度队列](./SchedulingQueue.md)、[调度框架](./Framework.md)、[调度插件](./Plugin.md)等重大，但是对于了解[调度器](./Scheduler.md)从生到死整个过程来说是必要一环，况且确实能学到点东西。

# Configurator

## Configurator定义

Configurator类似于[Scheduler](./Scheduler.md)的工厂，但是一般的工厂类很少会有这么多的配置，所以Configurator这个名字比Factroy更合适。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/factory.go#L59>

```go
// Configurator根据配置构造调度器
type Configurator struct {
	// 因为kube-scheduler需要访问apiserver，所以clientset.Interface是必须的。
	client clientset.Interface

	recorderFactory profile.RecorderFactory

	// informerFactory和client都是用来访问apiserver，informerFactory用来读API对象，client用于写API对象(比如绑定)
	informerFactory informers.SharedInformerFactory

	// 这个关闭信号，用chan实现是非常普遍的做法
	StopEverything <-chan struct{}

	// 调度缓存，调度器必须的模块，这可以证明调度缓存不是Scheduler构造的，而是通过Configurator传给Scheduler
	schedulerCache internalcache.Cache

	// 是否运行所有的FilterPlugin，即便中间某个插件返回失败。
	// 了解调度插件的读者都知道，有任何过滤插件返回失败，表示Pod不可调度，所有的过滤插件是与的关系。
	// 所以排序靠前的过滤插件返回失败理论上是无需再用后面的插件过滤了，这个配置就是是否运行所有的过滤插件。
	alwaysCheckAllPredicates bool

	// 这几个配置参数已经在调度器配置、调度框架、调度队列等文章中详细解释过了，Configurator只需要透明传递就好了
	percentageOfNodesToScore int32
	podInitialBackoffSeconds int64
	podMaxBackoffSeconds int64

	// 每个KubeSchedulerProfile对应一个调度框架(Framework)的配置，所以Configurator需要根据配置构造Framework
	profiles          []schedulerapi.KubeSchedulerProfile
	// 调度插件工厂注册表，即根据调度插件名字与调度插件工厂的映射。
	// 因为Configurator根据配置为Framework创建插件，所以需要根据插件名字获取调度插件工厂来构造插件
	registry          frameworkruntime.Registry
	// 调度缓存快照，熟悉调度缓存的读者应该知道，Scheduler每次调度之前都会更新调度缓存快照，调度缓存只需要将更新与快照diff的部分即可。
	// 所以调度缓存快照是和Scheduler一起创建，生命周期与调度器是相同的，由Configurator创建再传递给Scheduler。
	nodeInfoSnapshot  *internalcache.Snapshot
	// 调度扩展程序配置，Configurator需要根据配置构造调度扩展程序并传递给Scheduler
	extenders         []schedulerapi.Extender
	// frameworkCapturer是回调函数，用来捕获每最终的KubeSchedulerProfile，目的是什么？
	// 因为输入的KubeSchedulerProfile里包含有使能和禁止的插件配置，而Configurator构造Scheduler时会合并出最终的KubeSchedulerProfile。
	// 如果需要获取最终的KubeSchedulerProfile怎么办？就可以通过FrameworkCapturer捕获。
	frameworkCapturer FrameworkCapturer
	// 用来配置最大并行度，这个参数再调度器配置中有详细说明，说的简单点就是配置最大协程的数量。
	parallellism      int32
}
```

## create

直接进入Configurator最核心的函数，就是更具配置构造Scheduler，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/factory.go#L90>

```go
// create()创建Scheduler对象
func (c *Configurator) create() (*Scheduler, error) {
	var extenders []framework.Extender
	var ignoredExtendedResources []string
	// 是否配置了Extender?
	if len(c.extenders) != 0 {
		var ignorableExtenders []framework.Extender
		// 遍历所有的Extender配置
		for ii := range c.extenders {
			// 根据配置构造Extender(当前只有HTTPExtender实现)
			klog.V(2).InfoS("Creating extender", "extender", c.extenders[ii])
			extender, err := core.NewHTTPExtender(&c.extenders[ii])
			if err != nil {
				return nil, err
			}
			// 根据Extender'是否可以被忽略'将Extender分为两个集合:extenders和ignorableExtenders
			if !extender.IsIgnorable() {
				extenders = append(extenders, extender)
			} else {
				ignorableExtenders = append(ignorableExtenders, extender)
			}
			// 遍历Extender管理的资源，统计Scheduler可以忽略的资源
			for _, r := range c.extenders[ii].ManagedResources {
				if r.IgnoredByScheduler {
					ignoredExtendedResources = append(ignoredExtendedResources, r.Name)
				}
			}
		}
		// 最后将ignorableExtenders追加到extenders尾部，这样做的目的笔者在解析Extender的文章中有说明。
		// 此处简单描述一下：在遍历Extender调度Pod时，因为二者存在明确的分界线，当调用错误时可以根据当前Extender是否可忽略选择返回错误(否)或者成功(是)
		extenders = append(extenders, ignorableExtenders...)
	}

	// 如果Extender中有任何资源需要Scheduler忽略掉，需要通过追加到每个Profile的PluginConfig一个忽略种资源的参数。
	// 这仅对v1beta1有效，可以在配置Extender和插件参数，对于较早的版本，不允许同时使用策略(Policy)和自定义插件配置。
	// 这非常有意思，没想到还可以这么玩儿，这样在接下来构造调度插件的时候就可以把忽略的资源通过参数告知调度插件。
	// 笔者在解析调度插件的文章中提到了noderesources.Fit这个插件，专门负责资源匹配的插件，所以忽略哪些资源应该告知这个插件。
	if len(ignoredExtendedResources) > 0 {
		for i := range c.profiles {
			prof := &c.profiles[i]
			pc := schedulerapi.PluginConfig{
				Name: noderesources.FitName,
				Args: &schedulerapi.NodeResourcesFitArgs{
					IgnoredResources: ignoredExtendedResources,
				},
			}
			prof.PluginConfig = append(prof.PluginConfig, pc)
		}
	}

	// 构造PodNominator，所有的调度框架(Framework)共享使用。
	// 不可能每个Framework独享一个PodNominator，这会导致Pod1提名了Node1而在调度Pod2时全然不知，因为Pod1和Pod2可能使用不同的调度框架。
	nominator := internalqueue.NewPodNominator()
	// ClusterEvent抽象了系统资源状态是如何更改的，比如Pod的Added，感兴趣的读者可以看看ClusterEvent的定义。
	// clusterEventMap是系统资源状态的更改到插件名字集合的映射，说白了就是有哪些插件更改了资源，至于有什么用读者自己研究吧~
	clusterEventMap := make(map[framework.ClusterEvent]sets.String)
	// 根据[]KubeSchedulerProfile构造map[string]Framework(map查找更快)，传入了调度框架和调度插件需要的参数。
	// 如何根据KubeSchedulerProfile构造Framework，读者可以自己看一下，相对比较简单，笔者此处不再注释了。
	profiles, err := profile.NewMap(c.profiles, c.registry, c.recorderFactory,
		frameworkruntime.WithClientSet(c.client),
		frameworkruntime.WithInformerFactory(c.informerFactory),
		frameworkruntime.WithSnapshotSharedLister(c.nodeInfoSnapshot),
		frameworkruntime.WithRunAllFilters(c.alwaysCheckAllPredicates),
		frameworkruntime.WithPodNominator(nominator),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(c.frameworkCapturer)),
		frameworkruntime.WithClusterEventMap(clusterEventMap),
		frameworkruntime.WithParallelism(int(c.parallellism)),
	)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}
	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}
	// 此处需要注意了，使用第一个Profile的QueueSortPlugin的排序函数构造调度队列。
	// 我们知道调度队列只有一个，Shecuduler不会为每个调度框架创建一个调度队列，这就强行要求所有Profile必须配置同一个QueueSortPlugin。
	// 这一点笔者在解析调度器配置的文章中已经提到了，但是并没有从代码层面给出证明，此处算是证明了这一点。
	lessFn := profiles[c.profiles[0].SchedulerName].QueueSortFunc()
	podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		c.informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
		internalqueue.WithClusterEventMap(clusterEventMap),
	)

	// 与核心内容关系不大，本文忽略。
	debugger := cachedebugger.New(
		c.informerFactory.Core().V1().Nodes().Lister(),
		c.informerFactory.Core().V1().Pods().Lister(),
		c.schedulerCache,
		podQueue,
	)
	debugger.ListenForSignal(c.StopEverything)

	// 构造调度算法，笔者在解析调度算法的文章中提到了，调度算法的唯一实现就是genericScheduler
	algo := core.NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		extenders,
		c.percentageOfNodesToScore,
	)

	// 最终返回Scheduler对象
	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Profiles:        profiles,
		// 注入获取下一个待调度Pod的函数
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		// 注入调度Pod错误的处理函数
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		SchedulingQueue: podQueue,
	}, nil
}
```

虽然create()函数可以构造Scheduler，但是有没有发现缺少默认的插件配置？也就是说，create()直接利用Profile构造Framework，那么用户配置的Profile是怎么与默认的合并的呢？接下来的章节解析Configurator是如何生成默认配置并合并自定义配置。

## createFromProvider

kube-scheduler通过算法源获取插件的默认配置，算法源分为Provider和Policy两种，本章节介绍Provider，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/factory.go#L193>

```go
// createFromProvider()根据名字找到已注册的算法提供者以此构造Scheduler。
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).InfoS("Creating scheduler from algorithm provider", "algorithmProvider", providerName)
	// NewRegistry()返回了默认的插件配置(笔者在在解析调度插件的文章中介绍了默认插件配置)。
	// 感兴趣的读者可以了解一下这个函数，代码量不多，如果不了解的读者也没关系，就简单认为只有一种默认插件配置就可以了。
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

	// 遍历所有的Profile.
	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		// 在默认插件配置基础上应用用户的自定义配置，即使能新的插件或者禁止某些默认插件，相关函数读者自己查看即可。
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		// 将最终的插件配置更新到Profile中用于创建Framework，这个在前面已经提到了。
		prof.Plugins = plugins
	}
	// 创建Scheduler
	return c.create()
}
```

createFromProvider()就是利用默认的插件配置再合并用户自定义插件配置形成最终的插件配置，以此来创建Scheduler。有没有发现该方法是静态配置的，也就是已注册的Provider都是写在代码里的。在没有KubeSchedulerProfile这一功能之前，如果需要多种不同的Provider只能在代码里注册多个Provider，这明显是非常不友好的设计，并且无法为插件配置自定义参数，所以就有了基于策略(Policy)的插件配置。

## createFromConfig

基于策略(Policy)的插件配置可以通过配置文件、ConfigMap配置插件，这在KubeSchedulerProfile出来之间很好用，无奈KubeSchedulerProfile后来一同江湖了。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/factory.go#L213>

```go
// 从配置文件创建Scheduler，仅在v1alpha1可使用。新版本已经不推荐使用算法院(包括Provider和Policy)，改用KubeSchedulerProfile实现。
// 所以笔者不会对基于Policy构建Scheduler做详细解析，因为相对比较复杂且意义不大，只是简单介绍一下，感兴趣的读者可以自己详细看看。
func (c *Configurator) createFromConfig(policy schedulerapi.Policy) (*Scheduler, error) {
	// NewLegacyRegistry()创建基于策略的默认插件注册表，这一点与algorithmprovider.NewRegistry()功能类似。
	lr := frameworkplugins.NewLegacyRegistry()
	args := &frameworkplugins.ConfigProducerArgs{}

	klog.V(2).InfoS("Creating scheduler from configuration", "policy", policy)

	// 验证策略配置的有效性
	if err := validation.ValidatePolicy(policy); err != nil {
		return nil, err
	}

	// 根据策略配置找到所有已注册的Predicate的插件名字，Predicate在对应于当前的Filter
	predicateKeys := sets.NewString()
	if policy.Predicates == nil {
		klog.V(2).InfoS("Using predicates from algorithm provider", "algorithmProvider", schedulerapi.SchedulerDefaultProviderName)
		predicateKeys = lr.DefaultPredicates
	} else {
		for _, predicate := range policy.Predicates {
			klog.V(2).InfoS("Registering predicate", "predicate", predicate.Name)
			predicateName, err := lr.ProcessPredicatePolicy(predicate, args)
			if err != nil {
				return nil, err
			}
			predicateKeys.Insert(predicateName)
		}
	}

	// 根据策略配置找到所有已注册的Priority的插件名字，Priority在对应于当前的Score
	priorityKeys := make(map[string]int64)
	if policy.Priorities == nil {
		klog.V(2).InfoS("Using default priorities")
		priorityKeys = lr.DefaultPriorities
	} else {
		for _, priority := range policy.Priorities {
			if priority.Name == frameworkplugins.EqualPriority {
				klog.V(2).InfoS("Skip registering priority", "priority", priority.Name)
				continue
			}
			klog.V(2).InfoS("Registering priority", "priority", priority.Name)
			priorityName, err := lr.ProcessPriorityPolicy(priority, args)
			if err != nil {
				return nil, err
			}
			priorityKeys[priorityName] = priority.Weight
		}
	}

	// 设置Pod硬亲和权重参数
	if policy.HardPodAffinitySymmetricWeight != 0 {
		args.InterPodAffinityArgs = &schedulerapi.InterPodAffinityArgs{
			HardPodAffinityWeight: policy.HardPodAffinitySymmetricWeight,
		}
	}

	if policy.AlwaysCheckAllPredicates {
		c.alwaysCheckAllPredicates = policy.AlwaysCheckAllPredicates
	}

	klog.V(2).InfoS("Creating scheduler", "predicates", predicateKeys, "priorities", priorityKeys)

	// 因为在Policy中没有队列排序、抢占、绑定相关的配置，这些都用默认的插件
	plugins := schedulerapi.Plugins{
		QueueSort: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: queuesort.Name}},
		},
		PostFilter: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultpreemption.Name}},
		},
		Bind: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultbinder.Name}},
		},
	}
	// 基于策略配置生产插件配置
	var pluginConfig []schedulerapi.PluginConfig
	var err error
	if plugins, pluginConfig, err = lr.AppendPredicateConfigs(predicateKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if plugins, pluginConfig, err = lr.AppendPriorityConfigs(priorityKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if pluginConfig, err = dedupPluginConfigs(pluginConfig); err != nil {
		return nil, err
	}
	// 以下就和Provider一样了
	for i := range c.profiles {
		prof := &c.profiles[i]
		prof.Plugins = &schedulerapi.Plugins{}
		prof.Plugins.Append(&plugins)
		prof.PluginConfig = pluginConfig
	}

	return c.create()
}
```

基于策略的配置无需修改代码可以动态的调整插件配置，也可以配置插件参数。并且基于策略配置方法多样化，支持ConfigMap和文件。这些都是比Provider友好的地方，但是由于调度框架的升级，原有Predicates/Priority的设计已经无法适应，使得相关的配置也被淘汰。不得不说，基于策略的配置有它的先进性，而且从KubeSchedulerProfile的设计来看也有借鉴它的地方，比如插件参数。

# 总结

1. 无论是基于Provider还是Policy，都是配置调度算法的方法，前者是静态的，后者是动态，都称之为算法源；
2. 在KubeSchedulerProfile出来之前，算法源是配置插件的唯一方法，Provider通过代码静态编译不是很友好，Policy虽然不用修改代码，可以用ConfigMap和文件配置，但是依然无法适应多Scheduler的需求；
3. 算法源配置已经不推荐使用了，改用KubeSchedulerProfile方法，Configurator将算法源作为默认的插件配置，在此基础上合并用户自定的配置生成最终的[]KubeSchedulerProfile；
4. 既然不推荐使用算法源，但是从Configurator只有createFromProvider()和createFromConfig()两种创建Scheduler的接口，Scheduler构造函数选择的是createFromProvider()，默认配置了"DefaultProvider"，这就是KubeSchedulerProfile所基于的默认配置。
