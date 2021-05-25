<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-25 21:15:04
 * @Description: 
-->

# 前言

[调度框架](./Framework.md)和[调度插件](./Plugin.md)将调度流程抽象为若干的阶段，每个阶段称之为一个扩展点，也就是用户可以在每个阶段扩展自定义的[调度插件](./Plugin.md)。由于[调度框架](./Framework.md)和[调度插件](./Plugin.md)相对比较复杂，如果将调度视为一种接口(interface)，那么输入应该是一个Pod和全量的Node，输出是为Pod分配的节点。至于接口的使用者就可以弱化[调度框架](./Framework.md)和[调度插件](./Plugin.md)的概念，这是一种友好且通用的抽象。

再者，Pod.Spec.SchedulerName可以选择调度框架来调度该Pod，虽然不同的Pod的调度框架有所不同，但是调用调度插件的流程大部分都是一致的，但是在[调度框架](./Framework.md)文章中并没有看到有相关的成员函数的实现。那么，将所有调度插件串接起来，形成最终的调度能力是由随实现的呢？

在kube-scheduler中，有一种接口叫做ScheduleAlgorithm，就是负责在Node中为Pod选择一个最合适的Node。至于如何选择，肯定需要[调度框架](./Framework.md)和[调度插件](./Plugin.md)的支持，可以从[调度算法接口](#ScheduleAlgorithm定义)定义看出一些端倪。本文引用源码为kubernetes的release-1.21分支。

# ScheduleAlgorithm

## ScheduleAlgorithm定义

ScheduleAlgorithm是一种接口(interface)，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L61>

```go
// ScheduleAlgorithm抽象了调度一个Pod的接口。
type ScheduleAlgorithm interface {
    // 利用指定的调度框架调度Pod，为什么没有Node？因为Node是通过Cache获取的，所以在创建ScheduleAlgorithm对象的时候会传入Cache。
	// 来看看核心接口参数都有哪些：
	// 1. context.Context: 这个很好理解，当kube-scheduler退出时，调度算法也要及时退出，而不是完成复杂的调度算法再退出，否则可能无法实现优雅退出；
	// 2. framework.Framework: 调度一个Pod必须使用Pod.Spec.SchedulerName指定的调度框架，如果未指定使用默认的调度框架；
	// 3. *framework.CycleState: CycleState笔者在调度插件的文章中有详细说明，此处简单说明一下，调度一个Pod分为调度周期和绑定周期，
	//							 CycleState用于存储调度一个Pod的整个周期的状态，笔者更喜欢称Cycle为轮，调度一个Pod就是一轮调度。
	// 							 CycleState就是存储这一轮调度的各种状态，因为调度插件本身无状态，所以由调用者创建并在调度插件中共享使用。
	// 4. *v1.Pod: 这个很好理解，毕竟调度的就是一个Pod。
	// 为什么没有Node参数？否则调度算法怎么为Pod选择最优的Node？了解调度缓存的读者应该知道，Node相关信息是存储在调度缓存中。
	// 调度算法接口参数没有Node，唯一的解释就是调度算法对象有调度缓存亦或是调度缓存的快照，这样就不用每次调用都携带相同的参数了。
	// 还有，只有调度框架，为什么没有调度扩展程序(Extender)？因为调度扩展程序也是参与到调度Pod过程中的，唯一的解释与Node相同，
	// 即存储在调度算法对象中，这样不用每次都传递了，毕竟当前的设计调度扩展程序配置是相对静态的，一旦kube-scheduler启动就不会变化。
	Schedule(context.Context, framework.Framework, *framework.CycleState, *v1.Pod) (scheduleResult ScheduleResult, err error)
	// 这个接口只用来测试，所以不用关心
	Extenders() []framework.Extender
}

// ScheduleResult是调度一个Pod的结果，它包含最终选择的Node以及一些过程结果。
type ScheduleResult struct {
	// 最终选择的Node名字，也即是Pod即将调度到的Node
	SuggestedHost string
	// 调度过程中评估的Node数量
	EvaluatedNodes int
	// 本文需要引入一个概念：可行的(Feasible)Node，就是通过所有FilterPlguin的Node。FeasibleNodes就是可行Node的数量
	FeasibleNodes int
}
```

## genericScheduler

genericScheduler是ScheduleAlgorithm的一个实现(当前来看应该是唯一实现)，实现了调度一个Pod的普遍流程，此处用的‘普遍’就是想表达利用调度框架调度一个Pod的普遍流程，这也是用generic的目的吧。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L79>

```go
type genericScheduler struct {
    // 调度缓存，有了cache就有了所有的Node，这一点证明了笔者在解析ScheduleAlgorithm.Schedule()接口是的推断。
	cache                    internalcache.Cache
    // 所有调度扩展程序，这一点证明了笔者在解析ScheduleAlgorithm.Schedule()接口是的推断。
	extenders                []framework.Extender
    // 是否了解Cache.UpdateSnapshot(), 如果不了解请阅读https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Cache.md
    // 每调度一个Pod，都需要调用Cache.UpdateSnapshot()来更新本地的nodeInfoSnapshot。
	nodeInfoSnapshot         *internalcache.Snapshot
    // 这个变量用来提升调度性能的，percentageOfNodesToScore是一个百分比，PN/TN*100>=percentageOfNodesToScore就可以进入评分阶段。
    // 其中PN是可行Node数量，TN是总共Node数量。percentageOfNodesToScore在Node数量比较多的情况下减少评分阶段的计算量。
    // 比如一共有1000个Node，如果1000个都是可行Node，那么就需要计算1000个Node的分数，这可能是一个比较大的计算量。
    // 真的需要1000个Node都参与评分么？不一定，可能100个就够了，有的时候不需要找到全局的最优解，局部最优也是不错的，只要这个局部数量别太少。
	percentageOfNodesToScore int32
    // 就是因为percentageOfNodesToScore的存在，可能只要一部分Node参与调度。
    // 那么下次调度的时候希望不要一直在同一批Node上调度，直到有Node过滤失败再有新的Node加入进来。
    // nextStartNodeIndex就是用来记录下一次调度的时候从哪个Node开始遍历，每次调度完成后nextStartNodeIndex需要加上此次调度遍历Node的数量。
    // 这样调度下一个Pod的时候就从下一批Node开始，当然每次更新nextStartNodeIndex的时候需要考虑超出范围归零的问题。
	nextStartNodeIndex       int
}
```

## Schedule接口实现

了解了genericScheduler的定义，应该推测不出来调度一个Pod的大致实现。唯一可以确定的是，在调度前会调用一次Cache.UpdateSnapshot()更新快照，其他只能看具体源码实现了。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L97>

```go
// Schedule()实现了ScheduleAlgorithm.Schedule()接口.
func (g *genericScheduler) Schedule(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

    // snapshot()函数就是调用了g.cache.UpdateSnapshot(g.nodeInfoSnapshot)，更新Node快照，获取最新的Node状态
	if err := g.snapshot(); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

    // 如果没有Node，只能返回错误了。比如，当前集群还没有Node，此时创建的Pod在调度过程中产生的错误就是这里返回的。
	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

    // findNodesThatFitPod()找到Pod的可行Node，就是通过FilterPlugin(包括前处理)实现的。
	// findNodesThatFitPod()函数后续章节会详细解析。
	feasibleNodes, diagnosis, err := g.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")

    // 如果没有可行Node，返回错误，错误中包含了Node的数量以及每个Node过滤失败的结果，这些错误信息是为抢占调度提供参考的。
	// 一句话表达: 没有Node满足Pod的需求，唯一的可行性就是抢占。
	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: g.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}

	// 如果只有一个Node满足要求，那么该Node就是最优选，没有必要再通过评分寻找最优Node
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
			FeasibleNodes:  1,
		}, nil
	}

    // 'prioritize'是以前版本对于评分(Score)的称呼，就是给Node评分的过程。
	// 后续章节会有prioritizeNodes()详细解析。
	priorityList, err := g.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

    // 寻找最合适的Node，详情参看后续章节关于selectHost()的解析。
	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

    // 返回调度结果。
	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

如果对[调度框架](./Framework.md)和[调度插件](./Plugin.md)比较熟悉的话，那么上面的代码就非常好理解了。问：调度一个Pod总共分几步？答：分4步。
1. 准备：通过Cache.UpdateSnapshort()获取最新的Node的状态；
2. 过滤：过滤掉不满足Pod需求的Node；
3. 评分：对过滤通过的Node评分；
4. 选择：找到评分最高的Node就是最适合Pod的Node；

接下来，笔者对实现以上后三步的代码做详细的解析，因为第一步已经在[调度缓存](./Cache.md)中解析过了。

### 过滤

笔者在[调度插件](./Plugin.md)的文章中提到了，过滤分为[过滤前处理](./Plugin.md#PreFilterPlugin)、[过滤处理](./Plugin.md#FilterPlugin)和[过滤后处理](./Plugin.md#PostFilterPlugin)，那他们之间是什么样的逻辑关系形成最终的过滤功能？源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L223>

```go
// findNodesThatFitPod()找到所有满足Pod需求的Node，返回值framework.Diagnosis用来记录中间结果，感兴趣的读者可以看看类型定义。
// 当然，可以从本函数的代码可以猜出framework.Diagnosis的大部分功能。
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {
	// framework.Diagnosis包含了过滤每个Node的结果(NodeToStatusMap)以及哪些插件返回了不可调度。
	// 调度框架、调度插件、调度算法等模块称结果为状态，也对，状态包括成功、错误等。
	diagnosis := framework.Diagnosis{
		NodeToStatusMap:      make(framework.NodeToStatusMap),
		UnschedulablePlugins: sets.NewString(),
	}

	// 敲黑板！从现在开始就会感受到调度框架的魅力，因为有了调度框架，所以的代码感觉如此的顺其自然。
    // 运行所有的过滤前处理插件，为过滤准备数据。
	s := fwk.RunPreFilterPlugins(ctx, state, pod)
	// 获取所有的Node，此处是从Cache的快照中获取，此时是不是对快照的价值有一些认识了呢？
	// 正是因为有快照的存在，才使得每次获取的Node都是相同的，如果直接从Cache中列举每次可能都有不同。
	allNodes, err := g.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, diagnosis, err
	}
	if !s.IsSuccess() {
		// 如果不是不可调度的状态，那么按照错误返回。
        // 因为有些插件需要访问一些外部资源(即插件不可控的对象、函数等)，就会有返回错误的可能。
		if !s.IsUnschedulable() {
			return nil, diagnosis, s.AsError()
		}
		// 程序运行到这里说明Pod不可调度，那么问题来了，不是过滤的前处理么？怎么也有类似过滤的功能？
        // 笔者在此解释一下，过滤前处理是笔者对PreFilter的定义，主要功能是为Filter准备数据。
        // 但是不排除有些情况在因为无法获取准备的数据造成Pod暂时无法被调度，比如Pod声明的PVC不存在。
        // PreFilterPlugin实现的功能主要是为FilterPlugin准备数据而不是过滤，所有笔者更习惯称之为前处理。
		// 由于过滤前处理返回了不可调度状态，那么对于所有的Node来说，该Pod都是不可调度的。
		for _, n := range allNodes {
			diagnosis.NodeToStatusMap[n.Node().Name] = s
		}
		// 记录哪些返回不可调度状态
		diagnosis.UnschedulablePlugins.Insert(s.FailedPlugin())
		return nil, diagnosis, nil
	}

	// 由于抢占调度有可能在先前的调度周期中设置Pod.Status.NominatedNodeName。
	// 该Node可能是唯一适合该Pod的候选Node，因此，在遍历所有Node之前先尝试提名的Node。
	// 这个特性当前还不是发布状态，如果使用需要使能该特性。
	if len(pod.Status.NominatedNodeName) > 0 && feature.DefaultFeatureGate.Enabled(features.PreferNominatedNode) {
		// 评估提名的Node，至于怎么评估的，下面有evaluateNominatedNode()函数注释。
		feasibleNodes, err := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)
		if err != nil {
			klog.ErrorS(err, "Evaluation failed on nominated node", "pod", klog.KObj(pod), "node", pod.Status.NominatedNodeName)
		}
		// 提名的Node评估通过，应该将提名Node分配给Pod
		if len(feasibleNodes) != 0 {
			return feasibleNodes, diagnosis, nil
		}
	}
	// 利用FilterPlugin过滤Node，代码见下面注释
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, allNodes)
	if err != nil {
		return nil, diagnosis, err
	}

    // 对已经过滤的Node再用Extender过滤一遍，FilterPlugin.Filter()和Extender.Filter()是串联调用的.
    // 说明Pod必须同时满足FilterPlugin.Filter()和Extender.Filter()的过滤条件，两个过滤条件是'与'的关系。
    // 先用插件过滤再用Extender过滤，这样可以提前过滤掉不符合插件要求的Node，避免不必要的Node传输给Extender，这也是一种优化。
	// findNodesThatPassExtenders()代码下面有注释。
	feasibleNodes, err = g.findNodesThatPassExtenders(pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, diagnosis, err
	}
	return feasibleNodes, diagnosis, nil
}

// evaluateNominatedNode()评估提名Node是否满足Pod的需求。
func (g *genericScheduler) evaluateNominatedNode(ctx context.Context, pod *v1.Pod, fwk framework.Framework, state *framework.CycleState, diagnosis framework.Diagnosis) ([]*v1.Node, error) {
	// 根据提名Node的名字找到Node
	nnn := pod.Status.NominatedNodeName
	nodeInfo, err := g.nodeInfoSnapshot.Get(nnn)
	if err != nil {
		return nil, err
	}
	// 只将提名Node作为候选利用FilterPlugin过滤一下
	node := []*framework.NodeInfo{nodeInfo}
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, node)
	if err != nil {
		return nil, err
	}

	// 再利用调度扩展程序过滤一下
	feasibleNodes, err = g.findNodesThatPassExtenders(pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, err
	}

	// 评估提名Node其实就是看看提名Node是否满足过滤条件，感觉'评估'这个词有点大了，希望以后能扩展评估的范围吧...
	return feasibleNodes, nil
}

// findNodesThatPassFilters()利用FilterPlugin过滤Node
func (g *genericScheduler) findNodesThatPassFilters(
	ctx context.Context,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	diagnosis framework.Diagnosis,
	nodes []*framework.NodeInfo) ([]*v1.Node, error) {
	// numFeasibleNodesToFind()函数用来计算有多少Node通过过滤就可以了，还记得percentageOfNodesToScore这个成员变量么？
    // 换句话说就是如果通过过滤的Node太多会对后面的评分带来压力，但是太多并不会带来明显的提升，只要通过过滤的一部分就可以了。
    // numFeasibleNodesToFind()函数后面有注释。
	numNodesToFind := g.numFeasibleNodesToFind(int32(len(nodes)))

	// 创建过滤通过Node缓存，最多numNodesToFind个
	feasibleNodes := make([]*v1.Node, numNodesToFind)

    // 没有过滤插件等同于每个Node都能通过，所以直接返回连续的numNodesToFind的Node就可以了。
	if !fwk.HasFilterPlugins() {
		length := len(nodes)
		for i := range feasibleNodes {
			feasibleNodes[i] = nodes[(g.nextStartNodeIndex+i)%length].Node()
		}
		g.nextStartNodeIndex = (g.nextStartNodeIndex + len(feasibleNodes)) % length
		return feasibleNodes, nil
	}

	// 因为过滤每个Node彼此是无耦合的，所以可以用多协程对每个Node执行过滤操作，此处为并行操作做准备
    // 当并行运行的协程处理函数遇到错误可以将错误写入errCh，注意只有第一个写入成功，因为chan缓冲大小是1
	// 为什么不把chan缓冲设置为numNodesToFind？很简单，一个错误就可以终止Pod的调度，多了也没用。
	errCh := parallelize.NewErrorChannel()
	// 因为存在多协程并发写入framework.Diagnosis的可能，所以需要一个锁保护一下。
	// 需要注意的是framework.Diagnosis是1.21版本才引入的，之前是framework.NodeToStatusMap类型，所以锁的名字一直没有变。
	var statusesLock sync.Mutex
    // feasibleNodesLen用来记录当前已经过滤通过的Node数量，多协程之间共享，因为是原子操作它，所以不用加锁	
	var feasibleNodesLen int32
    // 为并发准备Context
	ctx, cancel := context.WithCancel(ctx)
	// checkNode是并行的协程函数，用来检测一个Node是否能够通过过滤，参数i是Node的索引
	checkNode := func(i int) {
		// 第i个协程过滤第i个Node，g.nextStartNodeIndex是本次调度周期的起始索引，这种操作应该比较常见不足为奇了吧
		nodeInfo := nodes[(g.nextStartNodeIndex+i)%len(nodes)]
		// 两个不同的地方会调用RunFilterPluginsWithNominatedPods函数：Schedule和Preempt(抢占)。
		// 当Schedule调用它时，要检测该Pod是否可以调度到该节Node上与已经存在的Pod加上提名到该节点的更高优先级或同等优先级的Pod一起运行。
		// 也就是说，不只是简单的过滤，同时还要考虑已经提名到该节点的Pod对该Pod的影响。
		status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
		// 如果过滤出错，则输出错误并返回
		if status.Code() == framework.Error {
			errCh.SendErrorWithCancel(status.AsError(), cancel)
			return
		}

		// 判断过滤是否通过
		if status.IsSuccess() {
			// Node过滤通过，则在feasibleNodesLen总数上加一
			length := atomic.AddInt32(&feasibleNodesLen, 1)
			if length > numNodesToFind {
				// 数量已经超过numNodesToFind，取消其他正在运行的协程
				cancel()
				// 总数减一，因为并没有输出到feasibleNodes中，这是很常规的原子操作使用方法
				atomic.AddInt32(&feasibleNodesLen, -1)
			} else {
				// 记录过滤通过的Node
				feasibleNodes[length-1] = nodeInfo.Node()
			}
		} else {
			// 过滤未通过，则记录该Node的状态，并且记录哪些插件过滤失败，过滤失败其实就是Pod不可调度。
			statusesLock.Lock()
			diagnosis.NodeToStatusMap[nodeInfo.Node().Name] = status
			diagnosis.UnschedulablePlugins.Insert(status.FailedPlugin())
			statusesLock.Unlock()
		}
	}

	beginCheckNode := time.Now()
	statusCode := framework.Success
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(runtime.Filter, statusCode.String(), fwk.ProfileName()).Observe(metrics.SinceInSeconds(beginCheckNode))
	}()

	// 启动多个协程并行过滤Node，此处需要注意的是并不是启动了len(allNodes)协程，parallelize包有一个并行度，默认值是16.
    // 如果并行度是16，那么最多启动16个协程，每个协程负责过滤一部分Node，就是典型的MapReduce模型。
    // 举个例子，如果总共有64个Node，那么每个协程负责过滤4个连续的Node。
	fwk.Parallelizer().Until(ctx, len(nodes), checkNode)
	// 总共过滤的Node数量是过滤失败的Node与成功的Node总和
	processedNodes := int(feasibleNodesLen) + len(diagnosis.NodeToStatusMap)
	// 计算下一个调度周期过滤Node的起始索引，就是当前的起始索引+过滤Node的总数，因为这些Node参与了当前Pod的过滤计算。
    // 看似没什么问题，其实这可能不是最好的方法：
	// 1. 过滤的Node里面有一些不满足Pod需求未通过，但是这些Node可能会满足下一个Pod的需求；
	// 2. 即便过滤通过的Node，最终只有一个Node会被选中，其他Node陪玩一轮，并且不会进入下一轮；
	// 3. 对于资源均匀分配的策略来说还好，但是对于资源需要集中分配的策略来说可能是个灾难；
	g.nextStartNodeIndex = (g.nextStartNodeIndex + processedNodes) % len(nodes)

    // feasibleNodes是按照最大量(numNodesToFind)申请的内存，实际可能没有那么多，所以需要调整到实际过滤通过的Node数量
	feasibleNodes = feasibleNodes[:feasibleNodesLen]
	// 如果过滤过程中有错误，则返回错误
	if err := errCh.ReceiveError(); err != nil {
		statusCode = framework.Error
		return nil, err
	}
	// 返回过滤成功的Node
	return feasibleNodes, nil
}

// numFeasibleNodesToFind()用来计算通过过滤Node的数量阈值，kube-scheduler认为通过过滤Node数量超过该阈值对调度效果影响不大，但是计算量会增大。
// 所以genericScheduler.percentageOfNodesToScore就是用来设置一个比例值，只选择Node总数一定比例的Node就可以了。
// 因为这个比例是配置的，有些情况可能不合理(比如Node总数不多)，所以需要需要结合实际情况计算。
func (g *genericScheduler) numFeasibleNodesToFind(numAllNodes int32) (numNodes int32) {
	// minFeasibleNodesToFind是一个常量(100)，也就是说无论比例是多少，最少也要100个Node，除非Node总数就不足100个。
    // g.percentageOfNodesToScore >= 100等于比例无效，无论多少Node都可以。
    // 这两种情况就不用计算，有多少来多少
	if numAllNodes < minFeasibleNodesToFind || g.percentageOfNodesToScore >= 100 {
		return numAllNodes
	}

    // 如果配置的比例<=0，需要计算自适应的比例
	adaptivePercentage := g.percentageOfNodesToScore
	if adaptivePercentage <= 0 {
		// 基础比例是50%
		basePercentageOfNodesToScore := int32(50)
		// Node总数满125个比例减一，可以持续累加，这个跟双十一满减一个原理，至于为什么是125笔者还没弄清楚。
        // 这可以理解，单纯的比例并不合理，尤其是Node总量非常大的情况下，比例应该适当调低。
        // 所以可以用函数y=(50x-0.008x^2)/100表示最终Node数量。初中数据告诉我们这是一个开口朝下的抛物线。
        // 这个函数有一个最大值，是当x=-b/2a，即Node总量为3125个时，应该选择782个Node。
        // 也就是说，当Node总量在[100(前面过滤了小于100的情况), 3125]时，选择Node的数量是在[49，782]范围单调递增的；
        // 当Node数量[3125, 7250]时，选择Node的数量是在[782, 0]单点递减的，超过7250比例就会变为负数。
		adaptivePercentage = basePercentageOfNodesToScore - numAllNodes/125
		// 比例总不能<=0吧？所以就有了minFeasibleNodesPercentageToFind(5)，即最小比例也应该是5%
		if adaptivePercentage < minFeasibleNodesPercentageToFind {
			adaptivePercentage = minFeasibleNodesPercentageToFind
		}
	}

	// 按照比例计算Node数量，adaptivePercentage = 50 - 0.008*numAllNodes
	// 所以numNodes = (50*numAllNodes - 0.008*numAllNodes^2) / 100，就是上面y=(50x-0.008x^2)/100的公式
	numNodes = numAllNodes * adaptivePercentage / 100
	// 最终还要限制在>=100范围，因为太少会影响调度效果
	if numNodes < minFeasibleNodesToFind {
		return minFeasibleNodesToFind
	}

	return numNodes
}

// findNodesThatPassExtenders()利用Extender过滤Node
func (g *genericScheduler) findNodesThatPassExtenders(pod *v1.Pod, feasibleNodes []*v1.Node, statuses framework.NodeToStatusMap) ([]*v1.Node, error) {
	// 遍历Extender, Extender是顺序调用的，而不是并发调用的。在一个Extender中排除一些可行Node，然后以递减的方式传递给下一个Extender。
	// 这种设计虽然可能降低Extender的传递Node的数据量，但是串行执行远程调用效率确实不可观。
	for _, extender := range g.extenders {
		// 可行的Node数量为0就没必要过滤了
		if len(feasibleNodes) == 0 {
			break
		}
		// Pod申请的资源不是Extender管理的也就不需要用它来过滤，这也是一种有效的提升效率的方法，尽量避免无效的远程调用。
		if !extender.IsInterested(pod) {
			continue
		}

		// 调用Exender过滤Node，了解Extender过滤详情参看笔者关于Extender的文章。
		feasibleList, failedMap, failedAndUnresolvableMap, err := extender.Filter(pod, feasibleNodes)
		if err != nil {
			// 如果Extender是可以忽略的，就会忽略Extender报错，也就是说调度Pod有没有该Extender参与都行，有自然更好。
			if extender.IsIgnorable() {
				klog.InfoS("Skipping extender as it returned error and has ignorable flag set", "extender", extender, "err", err)
				continue
			}
			return nil, err
		}

		// failAndUnresolvableMap中失败节点的状态会添加或覆盖到statuses。
		// 以便调度框架可以知悉特定节点的UnschedulableAndUnresolvable状态，这最终可以提高抢占效率。
		for failedNodeName, failedMsg := range failedAndUnresolvableMap {
			var aggregatedReasons []string
			if _, found := statuses[failedNodeName]; found {
				aggregatedReasons = statuses[failedNodeName].Reasons()
			}
			aggregatedReasons = append(aggregatedReasons, failedMsg)
			statuses[failedNodeName] = framework.NewStatus(framework.UnschedulableAndUnresolvable, aggregatedReasons...)
		}

		// 追加Extender过滤失败的原因
		for failedNodeName, failedMsg := range failedMap {
			if _, found := failedAndUnresolvableMap[failedNodeName]; found {
				// failedAndUnresolvableMap优先于failedMap，请注意，仅当Extender在两个map中都返回Node时，才会发生这种情况
				continue
			}
			if _, found := statuses[failedNodeName]; !found {
				statuses[failedNodeName] = framework.NewStatus(framework.Unschedulable, failedMsg)
			} else {
				statuses[failedNodeName].AppendReason(failedMsg)
			}
		}

		// Extender过滤通过的Node作为下一个Extender过滤的候选
		feasibleNodes = feasibleList
	}
	return feasibleNodes, nil
}
```

### 评分

过滤通过的Node又称之为可行Node，将作为评分阶段的候选Node列表，调度算法将利用ScorePlugin与Extender对每个可行Node进行评分。笔者将根据源码实现，解析调度算法的评分原理，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L406>

```go
// prioritizeNodes()为待调度的Pod给没给候选Node进行评分
func (g *genericScheduler) prioritizeNodes(
	ctx context.Context,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (framework.NodeScoreList, error) {
	// 既没有Extender也没有ScorePlugin，则所有节点的得分均为1。
	if len(g.extenders) == 0 && !fwk.HasScorePlugins() {
		result := make(framework.NodeScoreList, 0, len(nodes))
		for i := range nodes {
			result = append(result, framework.NodeScore{
				Name:  nodes[i].Name,
				Score: 1,
			})
		}
		return result, nil
	}

	// 运行所有PreScorePlugin准备评分数据
	preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		return nil, preScoreStatus.AsError()
	}

	// 运行所有ScorePlugin对Node进行评分，此处需要知道的是scoresMap的类型是map[string][]NodeScore。
    // scoresMap的key是插件名字，value是该插件对所有Node的评分
	scoresMap, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	if klog.V(10).Enabled() {
		for plugin, nodeScoreList := range scoresMap {
			for _, nodeScore := range nodeScoreList {
				klog.InfoS("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", plugin, "node", nodeScore.Name, "score", nodeScore.Score)
			}
		}
	}

	// result用于汇总所有分数。
	result := make(framework.NodeScoreList, 0, len(nodes))
	// 累加所有插件对于Node的评分
	for i := range nodes {
		result = append(result, framework.NodeScore{Name: nodes[i].Name, Score: 0})
		for j := range scoresMap {
			// 还记得在构建Framework的时候对配置权重的校验么？如果不做校验，那么此处就可能会溢出。
			// 需要注意的是，此处的分数是标准化分数并乘以插件对应的权重，相关的计算在调度框架中实现
			result[i].Score += scoresMap[j][i].Score
		}
	}

	// 如果配置了Extender，还要调用Extender对Node评分并累加到result中。
	if len(g.extenders) != 0 && nodes != nil {
		// 因为要多协程并发调用Extender并统计分数，所以需要锁来互斥写入Node分数
		var mu sync.Mutex
		var wg sync.WaitGroup
		// combinedScores的key是Node名字，value是Node评分
		combinedScores := make(map[string]int64, len(nodes))
		for i := range g.extenders {
			// 如果Extender不管理Pod申请的资源则跳过
			if !g.extenders[i].IsInterested(pod) {
				continue
			}
			// 启动协程调用Extender对所有Node评分。
			wg.Add(1)
			go func(extIndex int) {
				metrics.SchedulerGoroutines.WithLabelValues(metrics.PrioritizingExtender).Inc()
				defer func() {
					metrics.SchedulerGoroutines.WithLabelValues(metrics.PrioritizingExtender).Dec()
					wg.Done()
				}()
				// 调用Extender对Node进行评分
				prioritizedList, weight, err := g.extenders[extIndex].Prioritize(pod, nodes)
				if err != nil {
					// 此处有点意思，并没有返回错误，因为评分不同于过滤，对错误不敏感。过滤如果失败是要返回错误的(如果不能忽略)，因为Node可能无法满足Pod需求；
                    // 而评分无非是选择最优的节点，评分错误只会对选择最优有一点影响，但是不会造成故障。
					return
				}
				// 与其他插件对Node的评分进行合并
				mu.Lock()
				for i := range *prioritizedList {
					host, score := (*prioritizedList)[i].Host, (*prioritizedList)[i].Score
					if klog.V(10).Enabled() {
						klog.InfoS("Extender scored node for pod", "pod", klog.KObj(pod), "extender", g.extenders[extIndex].Name(), "node", host, "score", score)
					}
					// Extender的权重是通过Prioritize()返回的，其实该权重是人工配置的，只是通过Prioritize()返回使用上更方便。
					// 合并后的评分是每个Extender对Node评分乘以权重的累加和
					combinedScores[host] += score * weight
				}
				mu.Unlock()
			}(i)
		}
		// 等待所有协程退出，即便是多协程并发调用Extender，也会存在木桶效应，即调用时间取决于最慢的Extender。
        // 虽然Extender可能都很快，但是网络延时是一个比较常见的事情，更严重的是如果Extender异常造成调度超时，那么就拖累了整个kube-scheduler的调度效率。
		wg.Wait()
		for i := range result {
			// 最终Node的评分是所有ScorePlugin分数总和+所有Extender分数总和
			// 此处标准化了Extender的评分，使其范围与ScorePlugin一致，否则二者没法累加在一起。
			result[i].Score += combinedScores[result[i].Name] * (framework.MaxNodeScore / extenderv1.MaxExtenderPriority)
		}
	}

	if klog.V(10).Enabled() {
		for i := range result {
			klog.InfoS("Calculated node's final score for pod", "pod", klog.KObj(pod), "node", result[i].Name, "score", result[i].Score)
		}
	}
	return result, nil
}
```

### 选择

对Node评分不是目的，只是一种手段，最终目的是选出最优的Node。如何选择最优，我们立刻能想到的就是分最高的Node，那么问题来了，如果最高分数相同怎么办？来看看调度算法的实现，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/core/generic_scheduler.go#L154>

```go
// selectHost()根据所有可行Node的评分找到最优的Node
func (g *genericScheduler) selectHost(nodeScoreList framework.NodeScoreList) (string, error) {
	// 没有可行Node的评分，返回错误
	if len(nodeScoreList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}
	// 在nodeScoreList中找到分数最高的Node，初始化第0个Node分数最高
	maxScore := nodeScoreList[0].Score
	selected := nodeScoreList[0].Name
	// 如果做高分数相同怎么办？genericScheduler的解决方法是先统计数量(cntOfMaxScore)
	cntOfMaxScore := 1
	for _, ns := range nodeScoreList[1:] {
		// 如果分数比候选的分数高则成为候选
		if ns.Score > maxScore {
			maxScore = ns.Score
			selected = ns.Name
			cntOfMaxScore = 1
		} else if ns.Score == maxScore {
			// 分数相同就累计数量
			cntOfMaxScore++
			// 有1/cntOfMaxScore的概率成为最优Node，毕竟大家分数都相同，选中谁就看命运了，当然越靠前概率越高。
			// 为什么越靠前概率越高，因为越往后数量越多，概率自然就越小，比如第二个同分的Node就有50%的概率会成为最优Node
			if rand.Intn(cntOfMaxScore) == 0 {
				selected = ns.Name
			}
		}
	}
	return selected, nil
}
```

# 总结

1. 只有同时满足FilterPlugin和Extender的过滤条件的Node才是可行Node，调度算法优先用FilterPlugin过滤，然后在用Extender过滤，这样可以尽量减少传给Extender的Node数量；
2. 调度算法为待调度的Pod对每个可行Node(过滤通过)进行评分，评分方法是$\sum^n_0f(ScorePlugin_i)*w_i+\sum^m_0g(Extender_j)*w_j$，其中f(x)和g(x)是标准化分数函数，w为权重；
3. 分数最高的Node为最优候选Node，当有多个Node都为最高分数时，每个Node有1/n的概率成最优Node，其中n是可行Node列表中第几个最高分数的Node，例如：第一个分数最高的Node成为最优的Node的概率是1/1，第二个是1/2，第三个是1/3...，以此类推；
4. 调度算法并不是对[调度框架](./Framework.md)和[调度插件](./Plugin.md)再抽象和封装，只是对调度周期从PreFilter到Score的过程的一种抽象，其中PostFilter不在调度算法抽象范围内。因为PostFilter与过滤无关，是用来实现抢占的扩展点；
