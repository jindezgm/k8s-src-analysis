<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-11 19:56:43
 * @Description: 调度框架扩展源码解析
-->

# 前言

kube-scheduler基于Kubernetes管理的资源（比如CPU、内存、PV等）进行调度，但是，当需要对在Kubernetes外部管理的资源进行调度时，在Extender之前没有任何机制可以做到这一点。有一种方法可以让kubernetes具有扩展性，就是增加HTTP调度扩展程序。

有三种方法可以向kube-scheduler添加新的调度规则：

1. 更新现有的或添加新的调度插件并重新编译，这个在[调度插件](./Plugin.md)和[调度框架](./Framework.md)可以找到答案；
2. 开发一个自定义的kube-scheduler，该进程可代替标准kube-scheduler运行或与之并行运行;
3. 开发一个调度扩展程序(scheduler extender)程序，kube-scheduler在做调度决策的过程中调用扩展程序一起决策；

第一种和第二种方法对初学者有很高的要求，调度插件必须用go编写，并与kube-scheduler一起编译。开发一个自定义的kube-scheduler如果需要支持所有功能（例如扩展，亲和力，污点），则需要大量的工作量。第三种方法虽然也有缺点，例如性能低下和缓存不一致，但它也有几个优点：

1. 无需重新编译kube-scheduler就可以扩展现有调度程序的功能;
2. 调度扩展程序可以用任何语言编写；
3. 一旦实现调度扩展程序，可以用于扩展不同版本的kube-scheduler；

本文介绍第三种方法，如果不关心延迟和缓存一致性，并且不想维护自定义构建的kube-scheduler，那么这种方法可能是更好的选择。如果关心性能并希望最大程度地自定义kube-scheduler，那么开发新调度插件可能是更好的选择。

本文参考了[调度扩展程序官方设计文档](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/1819-scheduler-extender)，感兴趣的读者可以阅读原文。本文引用源码为kubernetes的release-1.21分支。

# Extender

## Extender接口

对于kube-scheduler，需要定义调度扩展程序的接口，就是Extender接口，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/extender.go#L27>

```go
type Extender interface {
    // 调度扩展程序的唯一名字，因为可能会有多个调度扩展程序。
	Name() string

    // Filter()和FilterPlugin.Filter()类似，不同的是传入了全部的Node，而插件传入的是一个Node，这个是出于调用效率考虑的，毕竟是远程调用。
	// 因为参数与过滤插件不同，所以返回值也略有不同，返回了已过滤的Node(通过过滤)和过滤失败(未通过过滤)的Node。
	Filter(pod *v1.Pod, nodes []*v1.Node) (filteredNodes []*v1.Node, failedNodesMap extenderv1.FailedNodesMap, err error)

	// Prioritize()这接口名字是有历史原因的，因为以前的调度器分为‘predicate’和‘prioritize’两个阶段，对应调度插件的Filter和Score。
	// Prioritize()接口要求输入Pod以及Filter()返回的Node集合，输出所有Node的评分(hostPriorities)以及调度扩展程序的权重。
	// 这样kube-scheduler就可以将所有扩展程序返回的分数乘以权重再累加起来，这一点和调度插件原理是一样的。
	Prioritize(pod *v1.Pod, nodes []*v1.Node) (hostPriorities *extenderv1.HostPriorityList, weight int64, err error)

	// Bind()与BindPlugin.Bind()功能一样，只是参数的差异，了解DefaultBinder.Bind()读者应该知道，该函数最终将接口参数转换成了v1.Binding类型在执行绑定的。
	Bind(binding *v1.Binding) error

	// 告诉kube-scheduler调度扩展程序是否有绑定能力，如果有绑定能力kube-scheduler会优先用调度扩展程序绑定。
	// 需要注意: kube-scheduler会优先用调度扩展程序绑定还有一个条件，那就是Pod有些资源是由Extender管理。
	IsBinder() bool

	// 判断Pod是否有任何资源是被Extender管理的，因为有资源被Extender管理交给它绑定才有意义，否则不如直接用默认的绑定插件。
	IsInterested(pod *v1.Pod) bool

	// Extender的抢占调度接口，传入待调度Pod，Node和被强占的Pod候选‘nodeNameToVictims’，key是node名字，value是node上被强占的Pod。
	// 有同学肯定会问，不是让Extender执行抢占调度么？哪来的Node和被强占的Pod候选？这些不应该是ProcessPreemption()返回的么？
	// 这是因为DefaultPreemption(唯一的抢占调度插件)在调用Extender.ProcessPreemption()之前已经执行了一部分抢占调度的来降低
	// Extender.ProcessPreemption()候选的数量，毕竟想要实现抢占调度既要满足调度插件的需求也要满足Extender的要求。
	// 所以先用调度插件选出一部分候选，可以减少不必要的数据传输，因为这是http调用。关于抢占调度的实现笔者会单独写一个文件解析。
	// ProcessPreemption()返回的结果可能：1）nodeNameToVictims的子集；2）候选Node上不同的被强占Pod集合
	ProcessPreemption(
		pod *v1.Pod,
		nodeNameToVictims map[string]*extenderv1.Victims,
		nodeInfos NodeInfoLister,
	) (map[string]*extenderv1.Victims, error)

	// 告知kube-scheduler是否具有抢占调度的能力。
	SupportsPreemption() bool

	// 告知kube-scheduler如果Extender不可用是否忽略，如果忽略，kube-scheduler不会返回错误。
	// 因为Extender的实现是HTTP服务，所以不可用是一种正常现象。
	IsIgnorable() bool
}
```

调度Pod时，扩展程序允许外部进程对节点进行过滤和评分(对应于[调度插件](./Plugin.md)的Filter和Score)。向调度扩展程序程序发出了两个独立的http/https调用，一个用于“过滤”操作，一个用于“评分”操作。如果无法调度Pod，则kube-scheduler将尝试抢占节点中优先级较低的Pod，并将其发送给调度扩展程序“preempt”动词（如果已配置）。调度扩展程序可以将Node和新的被强占Pod子集返回给调度程序。此外，调度扩展程序可以选择通过实现“绑定”操作将Pod绑定到apiserver。

## HTTPExtender

Extender是kube-scheduler抽象的调度扩展程序的接口，而调度扩展程序的实现是一个HTTP服务，也就是Extender的一些接口都对应的是远程调用。所以把Extender看做一个RPC也是可以的，既然是RPC，总要有客户端(Client)的实现，熟悉GRPC的读者应该都懂得哈~

此时就要引入HTTPExtender这个类型了，它实现了Extender，将Extender的一些接口转换为HTTP请求，所以是调度扩展程序的客户端。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/extender.go#L42>

```go
// HTTPExtender实现了Extender接口
type HTTPExtender struct {
	// 调度扩展程序的URL，比如https://127.0.0.1:8080。
	extenderURL      string
	// xxxVerb是HTTPExtender.Xxx()接口的HTTP请求的URL，比如https://127.0.0.1:8080/'preemptVerb' 用于ProcessPreemption()接口。
	preemptVerb      string
	filterVerb       string
	prioritizeVerb   string
	bindVerb         string
	// 调度扩展程序的权重，用来与ScorePlugin计算出最终的分数
	weight           int64
	// HTTP客户端
	client           *http.Client
	// 调度扩展程序是否缓存了Node信息，如果调度扩展程序已经缓存了集群中所有节点的全部详细信息，那么只需要发送非常少量的Node信息即可，比如Node名字。
	// 毕竟是HTTP调用，想法设法提升效率。但是为什么有podCacheCapable?这就要分析一下HTTPExtender发送的数据包括哪些了？
	// 1. 待调度的Pod
	// 2. Node(候选)
	// 3. 候选Node上的候选Pod（仅抢占调度)
	// 试想一下每次HTTP请求中Pod（包括候选Pod）可能不是不同的，而Node呢？有的请求可能会有不同，但于Filter请求因为需要的是Node全量，所以基本是相同。
	// 会造成较大的无效数据传输，所以当调度扩展程序能够缓存Node信息时，客户端只需要传输很少的信息就可以了。
	nodeCacheCapable bool
	// 调度扩展程序管理的资源名称
	managedResources sets.String
	// 如果调度扩展程序不可用是否忽略
	ignorable        bool
}
```

## [Extender配置](./KubeSchedulerConfiguration.md#Extender)

## Extender构造函数

Extender的配置与基本上与HTTPExtender一一对应，所以可以推测HTTPExtender的构造函数主要是通过配置赋值的过程，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/extender.go#L86>

```go
func NewHTTPExtender(config *schedulerapi.Extender) (framework.Extender, error) {
	// 没有配置超时，就用默认超时，5秒钟
	if config.HTTPTimeout.Duration.Nanoseconds() == 0 {
		config.HTTPTimeout.Duration = time.Duration(DefaultExtenderTimeout)
	}
	// 创建http.Client，makeTransport读者有兴趣自己看一下，跟大部分人用法一样
	transport, err := makeTransport(config)
	if err != nil {
		return nil, err
	}
	client := &http.Client{
		Transport: transport,
		Timeout:   config.HTTPTimeout.Duration,
	}
	// 管理的资源从slice转为map[string]struct{}
	managedResources := sets.NewString()
	for _, r := range config.ManagedResources {
		managedResources.Insert(string(r.Name))
	}
	// 各种通过配置赋值
	return &HTTPExtender{
		extenderURL:      config.URLPrefix,
		preemptVerb:      config.PreemptVerb,
		filterVerb:       config.FilterVerb,
		prioritizeVerb:   config.PrioritizeVerb,
		bindVerb:         config.BindVerb,
		weight:           config.Weight,
		client:           client,
		nodeCacheCapable: config.NodeCacheCapable,
		managedResources: managedResources,
		ignorable:        config.Ignorable,
	}, nil
}
```

要使用调度扩展程序，必须创建一个kube-scheduler配置文件，将上面的配置加进去。是不是感觉有点土？每次加一个调度扩展程序都要重启一下kube-scheduler，调度扩展程序将配置写入一个configmap，kube-scheudler监视(watch)配置，然后动态的添加、删除、更新它不香么？这个套路是笔者在实际项目中最常用的方法，保不齐哪天笔者会向社区提交一个可以动态配置调度扩展程序的PR！

## send

Kubernetes定义了各种请求(xxxVerb)的参数以及结果的类型，在<https://github.com/kubernetes/kube-scheduler/blob/release-1.21/extender/v1/types.go>中，本文就不一一列举了。所有的请求参数和结果都是序列化为JSON格式放在HTTP的Body中，这一点可以从send()函数看出来，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/extender.go#L413>

```go
func (h *HTTPExtender) send(action string, args interface{}, result interface{}) error {
	// 将请求参数(比如filter和prioritize请求是ExtenderArgs，preempt请求是ExtenderPreemptionArgs)序列化为JSON格式。
	out, err := json.Marshal(args)
	if err != nil {
		return err
	}
	// 格式化请求的最终URL
	url := strings.TrimRight(h.extenderURL, "/") + "/" + action
	// 创建HTTP请求（看来POST还是比较香的），并将JSON格式的参数放到Body中
	req, err := http.NewRequest("POST", url, bytes.NewReader(out))
	if err != nil {
		return err
	}
	// 内容格式是JSON
	req.Header.Set("Content-Type", "application/json")
	// 发送HTTP请求
	resp, err := h.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	// 检查HTTP的状态码，如果不是200就返回错误
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("failed %v with extender at URL %v, code %v", action, url, resp.StatusCode)
	}
	// 解析Body中的结果(比如filter请求是ExtenderFilterResult，prioritize请求是HostPriorityList，preempt请求是ExtenderPreemptionResult)
	return json.NewDecoder(resp.Body).Decode(result)
}
```

看到send函数的实现，基本可以推测Filter()、Prioritize()以及ProcessPreemption()的实现，但是笔者还是象征性的对代码做一下简单的注释。本文不对ProcessPreemption()做注释，因为笔者一直都在说有一个单独的文章分析kube-scheduler的抢占调度的实现，ProcessPreemption()的实现会放到那个文章中分析，感兴趣的读者自己可以看看。

## Filter实现

不废话了，直接上代码，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/extender.go#L275>

```go
func (h *HTTPExtender) Filter(
	pod *v1.Pod,
	nodes []*v1.Node,
) ([]*v1.Node, extenderv1.FailedNodesMap, error) {
	var (
		result     extenderv1.ExtenderFilterResult
		nodeList   *v1.NodeList
		nodeNames  *[]string
		nodeResult []*v1.Node
		args       *extenderv1.ExtenderArgs
	)
	// 将[]*v1.Node转为map[string]*v1.Node，当调度扩展程序缓存了Node信息，返回的结果只有Node名字。
	// fromNodeName用于根据Node名字快速查找对应的Node
	fromNodeName := make(map[string]*v1.Node)
	for _, n := range nodes {
		fromNodeName[n.Name] = n
	}
	// 如果没有配置filterVerb，说明调度扩展程序不支持Filter，所以直接返回
	if h.filterVerb == "" {
		return nodes, extenderv1.FailedNodesMap{}, nil
	}
	
	if h.nodeCacheCapable {
		// 如果调度扩展程序缓存了Node信息，则参数中只需要设置Node的名字
		nodeNameSlice := make([]string, 0, len(nodes))
		for _, node := range nodes {
			nodeNameSlice = append(nodeNameSlice, node.Name)
		}
		nodeNames = &nodeNameSlice
	} else {
		// 如果调度扩展程序没有缓存Node信息，就只能把全量的Node放在参数中
		nodeList = &v1.NodeList{}
		for _, node := range nodes {
			nodeList.Items = append(nodeList.Items, *node)
		}
	}
	// 构造HTTP请求参数
	args = &extenderv1.ExtenderArgs{
		Pod:       pod,
		Nodes:     nodeList,
		NodeNames: nodeNames,
	}
	// 发送HTTP请求
	if err := h.send(h.filterVerb, args, &result); err != nil {
		return nil, nil, err
	}
	if result.Error != "" {
		return nil, nil, fmt.Errorf(result.Error)
	}
	// 如果调度扩展程序缓存Node信息并且结果中设置了Node名字，那么前面[]*v1.Node转为map[string]*v1.Node就用上了
	if h.nodeCacheCapable && result.NodeNames != nil {
		nodeResult = make([]*v1.Node, len(*result.NodeNames))
		// 根据返回结果的Node名字找到Node并输出
		for i, nodeName := range *result.NodeNames {
			if n, ok := fromNodeName[nodeName]; ok {
				nodeResult[i] = n
			} else {
				return nil, nil, fmt.Errorf(
					"extender %q claims a filtered node %q which is not found in the input node list",
					h.extenderURL, nodeName)
			}
		}
	} else if result.Nodes != nil {
		// 直接从结果中获取Node
		nodeResult = make([]*v1.Node, len(result.Nodes.Items))
		for i := range result.Nodes.Items {
			nodeResult[i] = &result.Nodes.Items[i]
		}
	}

	return nodeResult, result.FailedNodes, nil
}
```

## Prioritize实现

不废话了，直接上代码，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/extender.go#L345>

```go
func (h *HTTPExtender) Prioritize(pod *v1.Pod, nodes []*v1.Node) (*extenderv1.HostPriorityList, int64, error) {
	var (
		result    extenderv1.HostPriorityList
		nodeList  *v1.NodeList
		nodeNames *[]string
		args      *extenderv1.ExtenderArgs
	)
	// 如果没有配置prioritizeVerb，说明调度扩展程序不支持Prioritize，所以直接返回
	if h.prioritizeVerb == "" {
		result := extenderv1.HostPriorityList{}
		for _, node := range nodes {
			result = append(result, extenderv1.HostPriority{Host: node.Name, Score: 0})
		}
		return &result, 0, nil
	}

	// 这部分和Filter()的一样，不重复注释了
	if h.nodeCacheCapable {
		nodeNameSlice := make([]string, 0, len(nodes))
		for _, node := range nodes {
			nodeNameSlice = append(nodeNameSlice, node.Name)
		}
		nodeNames = &nodeNameSlice
	} else {
		nodeList = &v1.NodeList{}
		for _, node := range nodes {
			nodeList.Items = append(nodeList.Items, *node)
		}
	}
	// 构造HTTP请求参数
	args = &extenderv1.ExtenderArgs{
		Pod:       pod,
		Nodes:     nodeList,
		NodeNames: nodeNames,
	}
	// 发送HTTP请求
	if err := h.send(h.prioritizeVerb, args, &result); err != nil {
		return nil, 0, err
	}
	return &result, h.weight, nil
}
```

# 总结

1. Extender是kube-scheduler抽象的调度扩展程序接口，kube-scheduler在调度流程的适当地方调用Extender协助kube-scheduler完成调度；
2. Extender可以实现过滤、评分、抢占和绑定，主要用来实现Kubernetes无法管理的资源的调度；
3. HTTPExtender是Extender的一种实现，用于将Extender的接口请求转为HTTP请求发送给调度扩展程序，配置调度扩展程序通过kube-scheduler的配置文件实现；
