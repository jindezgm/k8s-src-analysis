<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-27 21:07:15
 * @Description: kube-scheduler配置解析
-->

# 前言

kube-scheduler如何配置[调度框架](./Framework.md)和[调度插件](./Plugin.md)中提到的插件（比如使能或禁止哪些插件？）？如何配置[调度队列](./SchedulingQueue.md)的排序函数以及退避时间？本文将详细解析kube-scheduler的配置，在kube-scheduler中，配置也是一种API对象（其实在Kubernetes中都是这种设计，万物皆可API化），它被定义在`k8s.io/kubernetes/pkg/scheduler/apis/config`包中。

本文引用源码为kubernetes的release-1.21分支。

# KubeSchedulerConfiguration

KubeSchedulerConfiguration从类型名称上可知是kube-scheduler的配置API，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/types.go#L49>

```go
type KubeSchedulerConfiguration struct {
    // K8S所有的API必须有的类型元信息，不用多余解释了。
	metav1.TypeMeta

    // 调度Pod的算法的最大并行度，必须大于0，默认为16(在后续章节中会介绍如何设置默认配置)。
	// 还记得调度框架文章中提到的parallelize.Until()，比如kube-scheduler采用多协程并发计算分数，最大并行度就是这个参数指定的。
	Parallelism int32

	// 调度算法源，包括调度调度算法提供者（Provider）和调度策略两种，后者又分为文件和ConfigMap两种源，笔者在调度插件的文章中简单介绍过。
	// 但是AlgorithmSource的方式已经不推荐使用了，取而代之的是KubeSchedulerProfile，下面会看到，所以本文不会对算法源做过多介绍。
	AlgorithmSource SchedulerAlgorithmSource

	// LeaderElection选举相关的配置，采用多个kube-scheduler选举leader的方式实现高可用。
	// 本文不解析该配置，因为这属于基础组件的配置，从包名子componentbaseconfig也可以看出来，它不专属于kube-scheduler.
	// 笔者会单独写一篇文章解析Kubernetes的Leader选举，并介绍如何把Kubernetes的Leader选举用在自己的项目中。
	LeaderElection componentbaseconfig.LeaderElectionConfiguration

	// ClientConnection指定了kubeconfig文件和与apiserver通信时要使用的客户端连接设置。
	// ClientConnection同样是基础组件配置之一（毕竟需要连接apiserver的组件太多了），本文不做解析。
	ClientConnection componentbaseconfig.ClientConnectionConfiguration
	// HealthzBindAddress是检查kube-scheduler健康状态使用的IP地址和端口，默认为0.0.0.0:10251
	// MetricsBindAddress是获取kube-scheduler监控指标的IP地址和端口，默认为0.0.0.0:10251。
	// 这两个地址都是kube-scheduler对外提供http接口服务的地址，可以检测服务健康度以及获取各种监控指标。
	HealthzBindAddress string
	MetricsBindAddress string

	// 调试相关的配置，属于基础组件配置，本文不做解析。
	componentbaseconfig.DebuggingConfiguration

	// 如果集群特别大，每次调度一个Pod都需要遍历所有的Node计算量是不是非常大？PercentageOfNodesToScore就是用来减少计算量的。
	// 只要超过PercentageOfNodesToScore指定百分比的Node可行，那么调度器就在这些Node中寻找最优节点，而不会再遍历其他Node。
	// 例如：如果集群大小为500个节点，并且此配置的值为30，则调度器一旦找到150个可行Node就会停止查找。
	// 所以说，kube-scheduler并不是为每个Pod计算全局最优的Node，而是局部最优。
	PercentageOfNodesToScore int32

	// 用于配置调度队列的初始退避时间和最大退避时间，单位是秒。
	// 详情参看https://github.com/jindezgm/k8s-src-analysis/tree/master/kube-scheduler/SchedulingQueue.md
	PodInitialBackoffSeconds int64
	PodMaxBackoffSeconds int64

    // Profiles是kube-scheduler支持的调度配置，每个KubeSchedulerProfile对应一个独立的调度器，并有一个唯一的名字。
    // Pod可以指定调度器名字选择调度器，如果未指定任何调度程序名字，则将使用“default-scheduler”配置进行调度。
	// 后文专门有一个章节介绍KubeSchedulerProfile，因为KubeSchedulerProfile是本文的重点内容。
	Profiles []KubeSchedulerProfile

	// 调度扩展程序的配置列表，每个Extender都包含如何与调度扩展程序进行通信的配置，这些调度扩展程序由所有KubeSchedulerProfile共享。
	// 因为Extender是本质就是在kube-scheduler外部扩展调度插件，所以在所有KubeSchedulerProfile共享是必须的。
	Extenders []Extender
}
```

既然定义为API对象，按照Kubernetes的习惯，API对象可以通过Rest接口、文件、ConfigMap等方式访问。KubeSchedulerConfiguration是通过文件的方式进行配置的，那么问题来了，配置文件路径kube-scheduler如何知道？很简单，通过命令行--config设置，如下所示(执行kube-scheduler -h)：

```base
Usage:
  kube-scheduler [flags]

Misc flags:

      --config string                                                                                                                                                             
                The path to the configuration file. The following flags can overwrite fields in this file:
                  --address
                  --port
                  --use-legacy-policy-config
                  --policy-configmap
                  --policy-config-file
                  --algorithm-provider
```

需要注意的是，虽然可以通过文件配置KubeSchedulerConfiguration，但是依然可以通过--address、--port、--use-legacy-policy-config等覆盖配置中的字段。下面是KubeSchedulerConfiguration文件的简单示例(附带笔者注释)：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
# Leader选举配置
leaderElection:
  # 使能Leader选举
  leaderElect: true
# apiserver客户端连接配置
clientConnection:
  # 指定了kubeconfig文件路径
  kubeconfig: /etc/kubernetes/scheduler.conf
profiles:
  # 调度器[0]配置，名字是'default-scheduler'
- schedulerName: default-scheduler
  # 插件配置
  plugins:
    # 禁用QueueSort扩展点所有默认插件，使能Test
    queueSort:
      enabled:
      - name: Test
      disabled:
      - name: "*"
	# PreFilter扩展点使能Test
    preFilter:
      enabled:
      - name: Test
	# Permit扩展点使能Test
    permit:
      enabled:
      - name: Test
	# Reserve扩展点使能Test
    reserve:
      enabled:
      - name: Test
	# PostBind扩展点使能Test
    postBind:
      enabled:
      - name: Test
  # 插件参数配置
  pluginConfig:
  # Test插件的参数
  - name: Test
    args:
      abcd: efg
```

有没有注意到配置文件中apiVersion字段是`kubescheduler.config.k8s.io/v1beta1`，而本文前面介绍的KubeSchedulerConfiguration是在`k8s.io/kubernetes/pkg/scheduler/apis/config`包中，按照apiVersion字段声明的版本，应该是`k8s.io/kubernetes/pkg/scheduler/apis/config/v1beta`包中的类型。都是KubeSchedulerConfiguration类型，二者有什么区别么？其实这样的设计在Kubernetes其他的组件随处可见，`k8s.io/kubernetes/pkg/scheduler/apis/config`中定义的都是内部类型，kube-scheduler使用的也都是内部类型；而`k8s.io/kubernetes/pkg/scheduler/apis/config/v1beta`定义的是v1beta版本的类型，其实使用者可以定义多个版本（比如v1alpha），但是最终还是要转换为内部类型使用，转换函数注册在converter中。说了这么多其实需要读者有一个初步的了解，暂时不需要深入理解，当然，笔者也会有相关的文章介绍这些内容。

刨除一些相对不重要的配置参数，笔者总结kube-scheduler的配置主要包括：

1. [调度框架](./Framework.md)与[调度插件](./Plugin.md)相关的配置由Profiles(KubeSchedulerProfile)实现；
2. [调度扩展程序](./Extender.md)相关的配置由Extenders实现；
3. [调度队列](./SchedulingQueue.md)的退避时间配置由PodInitialBackoffSeconds和PodMaxBackoffSeconds实现；
4. [调度算法](./ScheduleAlgorithm.md)(插件)最大并行度由Parallelism实现；

接下来，笔者就会进一步解析调度插件的配置以及调度扩展程序的配置，至于其他的（基本组件配置除外）要么非常简单要么在其他文章中介绍过了，本文就不在解析。

## KubeSchedulerProfile

随着群集中的工作负载变得越来越多样化(异构)，很自然它们有不同的调度需求。kube-scheduler运行不同的[调度框架](./Framework.md)的[插件](./Plugin.md)配置，将其称为Profile，并关联一个调度器名字。通过设置Pod.Spec.SchedulerName可以选择特定配置来调度Pod。如果未指定调度器名字，则采用默认配置进行调度。

集群运行各种工作负载，大致可分为服务（长期运行）和批处理作业（运行后完成）。一些用户可能选择只在集群中运行一类工作负载，因此他们可以提供适合其调度需求的合理配置。但是，用户可以选择在单个群集中运行更多异构工作负载，或者他们可以具有一组固定节点和一组自动缩放的节点，每个节点都需要不同的调度行为。

Pod可以使用诸如node/pod亲和性，容忍度甚至是pod拓扑之类的功能来影响调度决策。但是有两个问题：

* 单个kube-scheduler配置以权重分数方式无法适应所有类型，例如，默认配置包括寻求服务高可用性的分数。
* 工作负载的开发者需要了解集群的特性和分数的权重来影响他们的Pod的调度。

为了服务于此类异构类型的工作负载，某些集群管理员选择运行多个调度程序，无论这些调度程序是不同的二进制文件还是具有不同配置的kube-scheduler。但是此设置可能会导致多个调度程序之间竞争，因为它们在给定时间可能对集群资源有不同的看法。此外，更多的二进制文件需要更多的管理工作。相反，只有一个kube-scheduler运行多个配置具有运行多个调度器而不会遇到竞争的优点。

kube-scheduler的配置在v1alpha2版本引入了KubeSchedulerProfile，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/types.go#L109>

```go
// KubeSchedulerProfile是调度配置，一个KubeSchedulerProfile在kube-scheduler中对应一个调度器(准确的讲应该是一个Framework)，并为其定义了唯一名字。
type KubeSchedulerProfile struct {
    // SchedulerName是与此配置关联的调度程序的名称，如果SchedulerName与Pod.Spec.SchedulerName匹配，则将使用此配置调度Pod。
	SchedulerName string

    // Plugins(下面有注释)为每个扩展点指定使能或禁用的插件集合，使能的插件是除默认插件之外还应启用的插件，禁用的插件是应禁用的任何默认插件。
    // 至于默认插件有哪些，笔者在调度插件和调度框架的文章中有提到，本文不赘述。如果没有为扩展点指定使能或禁用的插件，那么将使用该扩展点的默认插件。
    // 如果指定了QueueSort插件，那么所有配置必须指定相同的QueueSort插件和PluginConfig（该插件的参数，见下文）。
	Plugins *Plugins

    // PluginConfig是每个插件的一组可选的自定义插件参数。未配置插件的配置参数等同于使用该插件的默认配置参数。
    // 这个应该比较好理解，每个插件都可能有一些自定的配置，调度框架是不可能抽象所有可能参数的，所以为每个插件提供配置自定义参数的接口。
	PluginConfig []PluginConfig
}

// Plugins包括调度框架的全部扩展点的配置，因为特别简单，笔者就不一一注释了。
type Plugins struct {
	// 调度队列排序扩展点的插件配置，下面的就不一一注释了
	QueueSort PluginSet
	PreFilter PluginSet
	Filter PluginSet
	PostFilter PluginSet
	PreScore PluginSet
	Score PluginSet
	Reserve PluginSet
	Permit PluginSet
	PreBind PluginSet
	Bind PluginSet
	PostBind PluginSet
}

// PluginSet为扩展点指定使能和禁用的插件，如果数组为nil或者长度为0，则将使用该扩展点的默认插件。
type PluginSet struct {
    // Enabled指定除默认插件外还应启用的插件，这些将在默认插件之后以此处指定的顺序调用。
	// 那么问题来了，除了默认插件外，其他的插件都有啥？其实这里使能的插件应该都是使用者自己开发的插件。
	// 毕竟kube-scheduler不可能集成所有场景的插件，还是需要使用者根据自己需求开发特定的插件。
	// 需要注意的是，新开发的插件需要注册到kube-scheduler中，笔者知道的注册方法有两种：
	// 1. 直接写在默认的插件配置中，这样就不用使能了；
	// 2. 通过NewSchedulerCommand()函数的参数传入；
	Enabled []Plugin
    // Disabled指定禁用的默认插件，当需要禁用所有默认插件时，提供仅包含一个“*”的数组。
	Disabled []Plugin
}

// Plugin指定插件名称及其权重，权重仅用于Score插件。
type Plugin struct {
	// 插件的名字
	Name string
	// 插件的权重
	Weight int32
}

// PluginConfig指定在初始化时传递给插件的参数，在多个扩展点调用的插件将被初始化一次，Args可以具有任意结构体。
type PluginConfig struct {
	// 插件名字，与Plugin.Name匹配
	Name string
	// Args定义初始化时传递给插件的参数，可以具有任意结构体。这很合理，没有人能够知道未来扩展的插件需要什么样的参数。
	Args runtime.Object
}
```

PluginConfig.Args的类型runtime.Object，表示它也应该是一种API，并注册在Scheme中，这样kube-scheduler才能解析成对象传给插件，如下[代码](https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/register.go#L38)所示：

```go
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&KubeSchedulerConfiguration{},
		&Policy{},
		// 以下是注册各种插件参数的类型，相关定义笔者可以自己看一下，本文不再扩展
		&DefaultPreemptionArgs{},
		&InterPodAffinityArgs{},
		&NodeLabelArgs{},
		&NodeResourcesFitArgs{},
		&PodTopologySpreadArgs{},
		&RequestedToCapacityRatioArgs{},
		&ServiceAffinityArgs{},
		&VolumeBindingArgs{},
		&NodeResourcesLeastAllocatedArgs{},
		&NodeResourcesMostAllocatedArgs{},
		&NodeAffinityArgs{},
	)
	scheme.AddKnownTypes(schema.GroupVersion{Group: "", Version: runtime.APIVersionInternal}, &Policy{})
	return nil
}
```

## Extender

笔者在[《kube-scheduler调度扩展程序解析》](./Extender.md)中详细解析了Extender，如果对调度扩展程序不了解的读者建议先阅读一下。[HTTPExtender](./Extender.md#HTTPExtender)(调度扩展程序的一种实现，kube-scheduler采用HTTP协议与调度扩展程序交互，当前来看是唯一实现)中有很多成员变量是配置的，比如extenderURL、xxxVerb、weight等，都是通过调度扩展程序配置实现的，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/types.go#L324>

```go
// HTTPExtender的配置，起这个名字感觉像是通用Extender的配置，没准哪天又冒出来一个GRPCExtender怎么办？
type Extender struct {
	// HTTPExtender服务端的URL，为什么是前缀(URLPrefix)？因为它是每个请求最终URL的前缀。
	URLPrefix string
	// XxxVerb对应HTTPExtender.xxxVerb，具体功能参看解析Extender的文档。
	FilterVerb string
	PreemptVerb string
	PrioritizeVerb string
	BindVerb string

	// 权重，因为调度扩展程序可能包含有ScorePlugin
	Weight int64
	// 是否采用HTTPS与调度扩展程序通信
	EnableHTTPS bool
	// 如果采用HTTPS通信，创建http.Client就需要用到TLS配置，不需要解释
	TLSConfig *ExtenderTLSConfig
	// HTTPTimeout指定调用调度扩展程序的超时时间，在过滤阶段如果超时将无法调度Pod，在评分阶段(Prioritize)超时会被忽略。
	// 从这个参数可以看出，虽然调度扩展程序通过HTTP协议扩展了调度器，并且不受限于语言，但是存在超时的风险，万不得已笔者还是建议直接扩展调度插件。
	HTTPTimeout metav1.Duration
	// 对应HTTPExtender.nodeCacheCapable，此处不重复解释了
	NodeCacheCapable bool
	// 调度扩展程序管理的资源，即哪些资源归调度扩展程序调度，此处是slice，而HTTPExtender是字符串set，简单转换一下就行了
	ManagedResources []ExtenderManagedResource
	// 对应HTTPExtender.ignorable。
	Ignorable bool
}
```

## SetDefaults_KubeSchedulerConfiguration

前面章节中一直在说默认配置，那么默认配置到是怎样的？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/cmd/kube-scheduler/app/options/options.go#L137>

```go
// newDefaultComponentConfig()用于创建默认的KubeSchedulerConfiguration。
func newDefaultComponentConfig() (*kubeschedulerconfig.KubeSchedulerConfiguration, error) {
    // 首先创建v1beta1版本的KubeSchedulerConfiguration对象，所以是版本化的配置。
	versionedCfg := kubeschedulerconfigv1beta1.KubeSchedulerConfiguration{}
    // 关于Debug相关的配置本文不做详细说明，感兴趣的读者可以自己了解
	versionedCfg.DebuggingConfiguration = *configv1alpha1.NewRecommendedDebuggingConfiguration()

    // 设置versionedCfg的默认值(Scheme为每个API对象提供了注册设置默认值函数的接口，并通过Default()接口调用)
	kubeschedulerscheme.Scheme.Default(&versionedCfg)
    // 将v1beta1版本的KubeSchedulerConfiguration转换为KubeSchedulerConfiguration
	// 因为kube-scheudler使用的是内部类型，而不是版本化的类型。
	cfg := kubeschedulerconfig.KubeSchedulerConfiguration{}
	if err := kubeschedulerscheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
		return nil, err
	}
	return &cfg, nil
}
```

newDefaultComponentConfig()感觉做点事，又感觉什么也没做！毕竟只是调用了注册的设置默认值函数，那么v1beta1.KubeSchedulerConfiguration的设置默认值的函数又是哪个？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/v1beta1/defaults.go#L44>

```go
// SetDefaults_KubeSchedulerConfiguration()用于设置KubeSchedulerConfiguration的默认值。
func SetDefaults_KubeSchedulerConfiguration(obj *v1beta1.KubeSchedulerConfiguration) {
    // 最大并行度默认值是16
	if obj.Parallelism == nil {
		obj.Parallelism = pointer.Int32Ptr(16)
	}

	// newDefaultComponentConfig()函数可知obj.Profiles是空的。
	// 沿着newDefaultComponentConfig()思路obj.Profiles追加了一个未被初始化的v1beta1.KubeSchedulerProfile对象。
	if len(obj.Profiles) == 0 {
		obj.Profiles = append(obj.Profiles, v1beta1.KubeSchedulerProfile{})
	}
	// 因为上面代码追加了一个没有初始化话的v1beta1.KubeSchedulerProfile对象，所以需要为这个配置设置一个名字。
	// 这个名字就是默认调度器的名字DefaultSchedulerName("default-scheduler")
	if len(obj.Profiles) == 1 && obj.Profiles[0].SchedulerName == nil {
		obj.Profiles[0].SchedulerName = pointer.StringPtr(v1.DefaultSchedulerName)
	}

	// HealthzBindAddress和MetricsBindAddress不是本文的重点，所以相关设置默认值的代码此处省略
	...

	// 默认的PercentageOfNodesToScore是0(config.DefaultPercentageOfNodesToScore)
	// 如果为0，调度算法会根据集群Node数量计算一个自适应的比例。
	if obj.PercentageOfNodesToScore == nil {
		percentageOfNodesToScore := int32(config.DefaultPercentageOfNodesToScore)
		obj.PercentageOfNodesToScore = &percentageOfNodesToScore
	}

	// 基础组件配置的默认值此处省略
	...

	// 默认的初识退避时间是1秒
	if obj.PodInitialBackoffSeconds == nil {
		val := int64(1)
		obj.PodInitialBackoffSeconds = &val
	}

	// 默认的最大退避时间是10秒
	if obj.PodMaxBackoffSeconds == nil {
		val := int64(10)
		obj.PodMaxBackoffSeconds = &val
	}
}
```

那么问题来了，有了设置默认值的函数，是如何注册到Scheme的呢？API对象的设置默认值的函数是通过code-generator自动生成的，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/v1beta1/zz_generated.defaults.go#L31>

```go
func RegisterDefaults(scheme *runtime.Scheme) error {
	...
	// 将SetObjectDefaults_KubeSchedulerConfiguration注册为v1beta1.KubeSchedulerConfiguration{}的设置默认值的函数
	scheme.AddTypeDefaultingFunc(&v1beta1.KubeSchedulerConfiguration{}, func(obj interface{}) {
		SetObjectDefaults_KubeSchedulerConfiguration(obj.(*v1beta1.KubeSchedulerConfiguration))
	})
	...
	return nil
}

// SetObjectDefaults_KubeSchedulerConfiguration最终调用了SetDefaults_KubeSchedulerConfiguration
func SetObjectDefaults_KubeSchedulerConfiguration(in *v1beta1.KubeSchedulerConfiguration) {
	SetDefaults_KubeSchedulerConfiguration(in)
}
```

虽然知道是如何注册KubeSchedulerConfiguration的设置默认值函数，但是kube-scheduler是在什么时候注册的呢？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/v1beta1/register.go#L37>

```go
// 在引入v1beta1包的时候就注册了addDefaultingFuncs()函数
func init() {
	localSchemeBuilder.Register(addDefaultingFuncs)
}
```

addDefaultingFuncs()函数定义在<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/apis/config/v1beta1/defaults.go#L39>

```go
// 太简单了，不用解释了吧。
func addDefaultingFuncs(scheme *runtime.Scheme) error {
	return RegisterDefaults(scheme)
}
```

经过一系列的源码解析，有没有发现v1beta1.KubeSchedulerConfiguration相关的代码组织在不同的包中，笔者整理如下：

1. v1beta1.KubeSchedulerConfiguration的类型被定义在`k8s.io/kube-scheduler/config/v1beta1`
2. v1beta1.KubeSchedulerConfiguration的deepcopy、defaults、conversion等被被生成在`k8s.io/kubernetes/pkg/scheduler/apis/config/v1beta`

# 总结

1. kube-scheduler通过KubeSchedulerProfile实现多个调度器配置，每个调度器都有一个名字，默认的调度器名字是"default-scheduler"，可以通过Pod.Spec.SchedulerName为Pod选择调度器；
2. 每个调度器配置(KubeSchedulerProfile)通过Plugins配置每个扩展点(QueueSort、PreFilter、Filter等)，每个扩展点可以通过PluginSet使能/禁止指定的插件，每个插件通过Plugin配置名字和权重(只在Score扩展点使用)；
3. 调度插件的自定义参数通过PluginConfig配置；
4. 使能配置是在默认插件基础上增加的插件，禁用配置是需要禁用指定的默认插件；
5. 所有的KubeSchedulerProfile必须指定相同的QueueSort插件，至于为什么，[Configurator](./Configurator.md)可以找到答案；




