<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-14 21:58:00
 * @Description: 调度框架代码解析
-->

# 前言

在阅读本文之前，建议先看看[调度框架官方文档](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)和[调度框架设提案](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md)，也有必要先看看[调度插件](./Plugin.md)，便于理解本文内容。

本文提取以官方文档中笔者认为比较重要的信息，并做了批注：

1. kubernetes-scheduler已添加了许多功能，这使得代码越来越大，逻辑也越来越复杂。一个复杂的调度程序很难维护，其错误也很难找到和修复，并且那些自己修改调度程序的用户很难赶上并集成新的版本更新。当前kube-scheduler提供了webhooks来扩展器功能(Extender)，但是能力有限。**调度器功能越来越复杂，很难维护，虽然有http接口的Extender扩展，但是使用非常受限并且效率不高(详情笔者会在解析[Extender](./Extender.md)的文章中介绍)，所以就有了插件化的调度框架**
2. 调度框架是kube-scheduler的一种插件化设计架构，他将一组新的‘插件’APIs添加到现有的调度器中，插件是在调度器中编译好的(没有动态库方式灵活)。这些APIs允许大多数调度功能以插件的形式实现，同时使调度‘核心’保持简单且易于维护。**这句话怎么理解呢？就是大部分的调度功能(比如亲和性、资源均匀调度)都是有插件实现的，而且调度器只需要按照调度框架设计的流程执行即可。**
3. 每次调度一个Pod都分为两个周期，即调度周期和绑定周期。调度周期为Pod选择一个Node，绑定周期将该决策应用于集群。调度周期是串行运行的，而绑定周期可能是同时运行的。如果确定Pod不可调度或者存在内部错误，则可以终止调度周期或绑定周期。Pod将返回调度队列并重试。**调度周期是单协程的，按阶段先后、插件配置顺序串行调用，绑定周期因为有存储写入操作，所以需要异步执行，否则调度效率就太低了。所谓决策应用于集群就是向apiserver写一个API对象，集群内需要watch这类对象的服务就能感知到**
4. 调度框架定义了一些扩展点，插件注册后在一个或多个扩展点被调用，这些插件中的一些改变调度决策，而另一些仅用于提供信息。**所谓扩展点就是插件接口，因为可以按需实现各种各样的插件，所以称之为‘扩展’；而‘点’就是从调度队列的排序一直到Pod绑定的后处理一共有11个节点(也称之为阶段phase)。至于有的改变决策，有的提供信息很好理解，比如，PreXxxPlugins都是提供信息的，而Xxx都是改变决策的。**

下图是调度框架的设计图，源于官方文档，结合图和[调度插件](./Plugin.md)，就会基本了解kubernetes的调度框架。

![Framework](./scheduling-framework-extensions.png)

本文结合官方文档与源码解析调度框架的实现，引用源码为kubernetes的release-1.20分支。

# 调度框架

## Handle

句柄的概念在C/C++语言中比较常用，一般都是某个对象的指针，出于安全考虑句柄也可以是一个键，然后把对象放在容器里（比如map）。调度框架的句柄Handle就是调度框架的指针（详情见后文），句柄是提供给[调度插件](./Plugin.md)(在插件注册表章节有介绍)使用的。因为调度插件的接口参数只包含PodInfo、NodeInfo、以及CycleState，如果调度插件需要访问其他对象只能通过调度框架句柄。比如绑定插件需要的Clientset、抢占调度插件需要的SharedIndexInformer、批准插件需要的[等待Pod](./WaitingPod.md)访问接口等等。

现在来看看调度框架的句柄是怎么定义的，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/interface.go#L515>

```go
type Handle interface {
    // SharedLister是基于快照的Lister，这个快照就是在调度Cache文章中介绍的NodeInfo的快照。
    // 此处不扩展对SharedLister进行说明，只要有调度Cache的基础就可以了，无非是NodeInfo快照基础上抽象新的接口。
    // 在调度周期开始时获取快照，保持不变，直到Pod完成'Permit'扩展点为止，在绑定阶段是不保证快照不变的。
    // 因此绑定周期中插件不应使用它，否则会发生并发度/写错误的可能性，它们应该改用调度Cache。
    SnapshotSharedLister() SharedLister

    // 此处需要说明一下WaitingPod，就是在'Permit'扩展点返回等待的Pod，需要Handle的实现提供访问这些Pod的接口。
    // 遍历等待中的Pod。
    IterateOverWaitingPods(callback func(WaitingPod))

    // 通过UID获取等待中的Pod
    GetWaitingPod(uid types.UID) WaitingPod

    // 拒绝等待中的Pod
    RejectWaitingPod(uid types.UID)

    // 获取Clientset，用于创建、更新API对象，典型的应用就是绑定Pod。
    ClientSet() clientset.Interface

    // 用于记录调度过程中的各种事件
    EventRecorder() events.EventRecorder

    // 当调度Cache和队列中的信息无法满足插件需求时，可以利用SharedIndexInformer获取指定的API对象。
    // 比如抢占调度插件需要获取所有运行中的Pod。
    SharedInformerFactory() informers.SharedInformerFactory

    // 获取抢占句柄，只开放给实现抢占调度插件的句柄，用来协助实现抢占调度，详情见后面章节注释。
    PreemptHandle() PreemptHandle
}
```

句柄存在的意义就是为插件提供服务接口，协助插件实现其功能，且无需提供调度框架的实现。虽然句柄就是调度框架指针，但是interface的魅力就在于使用者只需要关注接口的定义，并不关心接口的实现，这也是一种解耦方法。

## PreemptHandle

抢占句柄是专为抢占调度插件(PostFilterPlugin)设计的的接口，试想一下我们自己实现抢占调度需要抢占句柄提供什么能力：1)[PodNominator](./PodNominator.md)记录了所有提名(抢占成功但还在等待被强占Pod退出)的Pod，所以抢占句柄至少应该包含PodNominator接口供插件使用(比如获取已经提名的但是比当前Pod优先级低的Pod)；2)抢占调度插件需要能够运行Filter、Score等插件才能评估抢占是否成功。
接下来就看看抢占句柄的定义，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/interface.go#L552>

```go
type PreemptHandle interface {
    // PodNominator是必须的，他可以获取提名到Node的所有Pod
    PodNominator
    // PluginsRunner下面有注释，用来运行插件评估抢占是否成功并选择最优Node
    PluginsRunner
    // Extenders笔者会专门写一篇文章介绍它，此处只需要知道它的实现是一个HTTP的服务，用来扩展kubernetes无法管理的资源的调度能力。
    Extenders() []Extender
}

// PluginsRunner运行某些插件，用于抢占调度插件评估Pod是否可以通过抢占其他Pod的资源运行在Node上。
type PluginsRunner interface {
    // RunXxxPlugins按照配置的插件循序运行Xxx扩展点的插件。
    // 从接口定义来看只有过滤（包括前处理）和评分（包括前处理）的插件，非常合理。
    // 因为抢占实现逻辑和普通的调度一样，都是过滤后再评分，不同的地方无非是多了被强占Pod的处理。
    RunPreScorePlugins(context.Context, *CycleState, *v1.Pod, []*v1.Node) *Status
    RunScorePlugins(context.Context, *CycleState, *v1.Pod, []*v1.Node) (PluginToNodeScores, *Status)
    RunFilterPlugins(context.Context, *CycleState, *v1.Pod, *NodeInfo) PluginToStatus
    RunPreFilterExtensionAddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
    RunPreFilterExtensionRemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToRemove *v1.Pod, nodeInfo *NodeInfo) *Status
}
```

## Framework定义

前言已经详细说明了调度框架产生的原因以及相关的设计思想，此处直接上代码，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/interface.go#L412>

```go
type Framework interface {
    // 调度框架句柄，构建调度插件时使用
    Handle
    // 获取调度队列排序需要的函数，其实就是QueueSortPlugin.Less()
    QueueSortFunc() LessFunc

    // 有没有发现和PluginsRunner很像？应该说PluginsRunner是Framework的子集，也可以理解为Framework的实现也是PluginsRunner的实现。
    //Framework这些接口主要是为了每个扩展点抽象一个接口，因为每个扩展点有多个插件，该接口的实现负责按照配置顺序调用插件。
    // 大部分扩展点如果有一个插件返回不成功，则后续的插件就不会被调用了，所以同一个扩展点的插件间大多是“逻辑与”的关系。
    RunPreFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod) *Status
    RunFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) PluginToStatus
    RunPostFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
    RunPreFilterExtensionAddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
    RunPreFilterExtensionRemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
    RunPreScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
    RunScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) (PluginToNodeScores, *Status)
    RunPreBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunPostBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
    RunReservePluginsReserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunReservePluginsUnreserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
    RunPermitPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status

    // 如果Pod正在等待（PermitPlugin返回等待状态），则会被阻塞直到Pod被拒绝或者批准。
    WaitOnPermit(ctx context.Context, pod *v1.Pod) *Status
    
    // 用来检测Framework是否有FilterPlugin、PostFilterPlugin、ScorePlugin，没难度，不解释
    HasFilterPlugins() bool
    HasPostFilterPlugins() bool
    HasScorePlugins() bool

    // 列举插件，其中：
    // 1. key是扩展点的名字，比如PreFilterPlugin对应扩展点的名字就是PreFilter，规则就是将插件类型名字去掉Plugin；
    // 2. value是扩展点所有插件配置，类型是slice
    ListPlugins() map[string][]config.Plugin

    // 此处需要引入SchedulingProfile概念，SchedulingProfile允许配置调度框架的扩展点。
    // SchedulingProfile包括使能/禁止每个扩展点的插件，每个插件的权重(仅在Score扩展点有效)以及插件参数；
    // 这个接口是用来获取SchedulingProfile的名字，因为v1.Pod.Spec.SchedulerName就是指定SchedulingProfile名字来选择Framework。
    ProfileName() string
}
```

从Framework的接口定义基本可以推测它的画像：

1. Framework是根据SchedulingProfile创建的，是一对一的关系，Framework中插件的使能/禁止、权重以及参数都来自SchedulingProfile;
2. Framework按照调度框架设计图中的扩展点依次提供了运行所有插件的RunXxxPlugins()接口，扩展点插件的调用顺序也是通过SchedulingProfile配置的；
3. 一个Framework就是一个调度器，配置好的调度插件就是调度算法（可以从kubernetes/pkg/scheduler/algorithmprovider的包名看出当时设计者的想法），kube-scheduler按照固定的流程调用Framework接口即可，这样kube-scheduler就相对简单且易于维护；

此处用一个不太恰当的比喻：Framework类似于动态库的接口定义，每个SchedulingProfile对应一个实现了Framework接口的动态库，而SchedulingProfile名字就是动态库的名字。kube-scheduler按需加载动态库，然后按照Pod选择的Framework执行调度。这是C/C++视角的理解，虽然kube-scheduler的插件都是静态编译，然后通过配置动态生成，但是原理上是相同的，只要有助于理解就可以。

## Framework实现

### frameworkImpl

为什么称之为框架？因为他是固定的，差异在于配置导致插件的组合不同。也就是说，即便有多个SchedulingProfile，但是只有一个Framework实现，无非Framework成员变量引用的插件不同而已。来看看Framework实现代码，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/runtime/framework.go#L66>
