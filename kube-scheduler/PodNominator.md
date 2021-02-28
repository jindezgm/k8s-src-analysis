<!--
 * @Author: jinde.zgm
 * @Date: 2021-02-28 09:13:16
 * @Description: 
-->

# 前言

笔者在[《SchedulingQueue》](./SchedulingQueue.md)的文章中提到了PodNominator，因为调度队列继承了PodNominator，但是当时为了避免不必要的内容扩展，并没有对PodNominator做任何说明。本文将对PodNominator做较全面的解析，从它的定义、实现直到在kube-scheduler中的应用。

本文采用的Kubenetes源码的release-1.20分支。

# PodNominator的定义

首先需要了解PodNominator到底是干啥的，英文翻译就是Pod提名器，那什么又是提名呢？这个可以从Pod的API定义的部分注释得到结果。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/staging/src/k8s.io/api/core/v1/types.go#L3594>

```go
    // nominatedNodeName is set only when this pod preempts other pods on the node, but it cannot be
    // scheduled right away as preemption victims receive their graceful termination periods.
    // This field does not guarantee that the pod will be scheduled on this node. Scheduler may decide
    // to place the pod elsewhere if other nodes become available sooner. Scheduler may also decide to
    // give the resources on this node to a higher priority pod that is created after preemption.
    // As a result, this field may be different than PodSpec.nodeName when the pod is
    // scheduled.
    // +optional
    NominatedNodeName string `json:"nominatedNodeName,omitempty" protobuf:"bytes,11,opt,name=nominatedNodeName"`
```

简单总结源码注释就是当Pod抢占Node上其他Pod的时候会设置Pod.Status.NominatedNodeName为Node的名字。调度器不会立刻将Pod调度到Node上，因为需要等到被抢占的Pod优雅退出。到这里就可以了，其他内容属于调度相关的，我们只需要知道提名到底是干啥的就可以了。

现在知道提名就是可以将Pod调度到提名的Node上，但是需要等被抢占的Pod退出后腾出资源才能执行调度。而PodNominator就是记录哪些Pod获得Node提名，那么问题来了，Pod.Status.NominatedNodeName不是已经记录了么，还要PodNominator干什么呢？笔者先不回答这个问题，在总结的章节中会给出答案。

先来看看PodNominator的定义，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/interface.go#L562>

```go
type PodNominator interface {
    // 提名pod调度到nodeName的Node上
    AddNominatedPod(pod *v1.Pod, nodeName string)
    // 删除pod提名的nodeName 
    DeleteNominatedPodIfExists(pod *v1.Pod)
    // PodNominator处理pod更新事件
    UpdateNominatedPod(oldPod, newPod *v1.Pod)
    // 获取提名到nodeName上的所有Pod，这个接口是不是感觉到PodNominator存在的意义了？
    // 毕竟PodNominator有统计功能，否则就需要遍历所有的Pod.Status.NominatedNodeName才能统计出来
    NominatedPodsForNode(nodeName string) []*v1.Pod
}
```

从PodNominator的定义来看非常简单，感觉用map就可以实现了，事实上也确实如此。

# PodNominator的实现

PodNominator的实现在调度队列中，这其中的用意值得研究一下。咱们先看代码实现，在研究为什么在调度队列中实现。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L723>

```go
// 确实非常简单，用map加读写锁实现的
type nominatedPodMap struct {
    // nominatedPods的key是提名的Node的名字，value是所有的Pod，这就是为NominatedPodsForNode()接口专门设计的
    nominatedPods map[string][]*v1.Pod
    // nominatedPodToNode的key是Pod的UID，value是提名Node的名字，这是为增、删、改接口设计的
    // 为什么调度队列用NS+NAME，而这里用UID，其中可能有版本更新造成的历史原因，关键要看是否有用名字访问的需求。
    // 当然，当前版本调度队列的接口没有直接用NS+NAME访问Pod，但是不排除以前是有这个设计考虑的。
    // 以上只是笔者猜测，仅供参考，有哪位小伙伴有更靠谱的答案欢迎在留言区回复。
    nominatedPodToNode map[ktypes.UID]string
 
    sync.RWMutex
}
 
// nominatedPodMap一共就三个核心函数，add、delete、和UpdateNominatedPod
func (npm *nominatedPodMap) add(p *v1.Pod, nodeName string) {
    // 避免Pod已经存在先执行删除，确保同一个Pod不会存在两个实例，这个主要是为了nominatedPods考虑的，毕竟它的value是slice，没有去重能力。
    npm.delete(p)
 
    // NominatedNodeName()函数就是获取p.Status.NominatedNodeName,
    // 那么下面的代码的意思就是nodeName和p.Status.NominatedNodeName优先用nodeName，如果二者都没有指定则返回
    // 这里需要知道的是nodeName代表着最新的状态，所以需要优先，这一点在PodNominator应用章节会进一步说明
    nnn := nodeName
    if len(nnn) == 0 {
        nnn = NominatedNodeName(p)
        if len(nnn) == 0 {
            return
        }
    }
    // 下面的代码没什么难度，就是把Pod放到两个map中
    npm.nominatedPodToNode[p.UID] = nnn
    for _, np := range npm.nominatedPods[nnn] {
        if np.UID == p.UID {
            klog.V(4).Infof("Pod %v/%v already exists in the nominated map!", p.Namespace, p.Name)
            return
        }
    }
    npm.nominatedPods[nnn] = append(npm.nominatedPods[nnn], p)
}
 
func (npm *nominatedPodMap) delete(p *v1.Pod) {
    // 如果Pod不存在就返回
    nnn, ok := npm.nominatedPodToNode[p.UID]
    if !ok {
        return
    }
    // 然后从两个map中删除
    for i, np := range npm.nominatedPods[nnn] {
        if np.UID == p.UID {
            npm.nominatedPods[nnn] = append(npm.nominatedPods[nnn][:i], npm.nominatedPods[nnn][i+1:]...)
            if len(npm.nominatedPods[nnn]) == 0 {
                delete(npm.nominatedPods, nnn)
            }
            break
        }
    }
    delete(npm.nominatedPodToNode, p.UID)
}
 
func (npm *nominatedPodMap) UpdateNominatedPod(oldPod, newPod *v1.Pod) {
    npm.Lock()
    defer npm.Unlock()
    // 首选需要知道一个知识点，kube-scheduler什么时候会调用UpdateNominatedPod()?这个问题貌似应该是PodNominator应用章节的内容。
    // 为了便于理解下面的代码，笔者需要提前剧透一下，答案是在调度队列的更新接口中，感兴趣的同学可以回看《kube-scheduler的SchedulingQueue解析》的源码注释
    // 而调度队列的更新是kube-scheduler在watch apiserver的Pod的时候触发调用的，所以此时默认是没有提名Node的
    nodeName := ""
    // 有一些情况，在Pod刚好提名了Node之后收到了Pod的更新事件并且新Pod.Status.NominatedNodeName=""，此时需要保留提名的Node。
    // 以下几种情况更新事件是不会保留提名的Node
    // 1.设置Status.NominatedNodeName:表现为NominatedNodeName(oldPod) == "" && NominatedNodeName(newPod) != ""
    // 2.更新Status.NominatedNodeName:表现为NominatedNodeName(oldPod) != "" && NominatedNodeName(newPod) != ""
    // 3.删除Status.NominatedNodeName:表现为NominatedNodeName(oldPod) != "" && NominatedNodeName(newPod) == ""
    if NominatedNodeName(oldPod) == "" && NominatedNodeName(newPod) == "" {
        if nnn, ok := npm.nominatedPodToNode[oldPod.UID]; ok {
            // 这是唯一一种情况保留提名
            nodeName = nnn
        }
    }
    // 无论提名的Node名字是否修改都需要更新，因为需要确保Pod的指针也被更新
    npm.delete(oldPod)
    npm.add(newPod, nodeName)
}
```

除了更新保留原有提名Node部分稍微有点复杂，整个PodNominator的实现可以说简单到不能再简单，甚至可以毫不夸张的说，在看到PodNominator的定义的时候就可以想象得到它的实现了。虽然PodNominator看似非常简单，但是很多复杂的功能就是又这些简单的模块组建而成。PodNominator是kube-scheduler实现抢占调度的关键所在，此时是不是感觉他就没那么简单了呢？现在笔者就带着大家看看PodNominator在kube-scheduler是如何应用，进而实现抢占调度的。

# PodNominator应用

在kube-scheduler中，有两个类型直接继承了PodNominator，它们分别是SchedulingQueue和PreemptHandle。前者笔者在[《SchedulingQueue》](./SchedulingQueue.md)文章中已经提到了，但是直接忽略了，所以本文算是调度队列的一个延续；PreemptHandle是抢占句柄，本文不会过多介绍，还是老套路，留给后续文章解析，但是从类型名字可以知道是用来实现抢占的。所以说PodNominator与抢占调度是有关系的。

虽然说SchedulingQueue和PreemptHandle都继承了PodNominator，但是他们都指向了同一个对象(这也是golang语言的特点，指针继承)，所以kube-scheduler中只有一个PodNominator对象。这也是nominatedPodMap加锁的一个原因，因为有多协程并发访问的需求。

## PodNominator在调度队列中的应用

调度队列有三种情况会调用PodNominator.AddNominatedPod():

1. PriorityQueue.Add()：源码链接<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L265>，此处的目的是将添加的Pod如果提名了Node，那就添加到PodNominator中。笔者认为正常的逻辑可能不会出现这种情况，因为一个新建的Pod何来提名？还记得SharedInformer的ResyncPeriod么？SharedInformer会定时全量同步一次，此时的事件是Add，以上仅是笔者的猜测。
2. PriorityQueue.AddUnschedulableIfNotPresent()：源码链接<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L326>，此处的目的是恢复Pod的提名状态，因为该函数会将Pod加入到unschedulableQ或者backoffQ，均为不可调度状态，原有的提名需要删除，恢复到Pod.Status.NominatedNodeName。
3. PriorityQueue.Update()：源码链接<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/internal/queue/scheduling_queue.go#L460>，此处的目的与PriorityQueue.Add()相同，因为更新的时候如果Pod不在任何子队列就当Add处理。
调度队列在PriorityQueue.Update()函数中如果Pod已经在队列中存在，会调用PodNominator。UpdateNominatedPod(),相应的代码笔者不再拷贝了，读者可以通过上面的连接找到。毕竟Pod已经更新了，相应的状态也需要更细难道PodNominator中，就是更新Pod的指针。同理，在PriorityQueue.Delete()函数中调用了PodNominator.DeleteNominatedPodIfExists()，笔者就不再解释了。

## PodNominator在调度器中的应用

当kube-scheduler需要通过抢占的方式为Pod提名某个Node时，此时的Pod仍然处于未调度状态，因为Pod需要等到Node上被抢占的Pod退出。所以此时对于Pod而言是调度失败的，也就是PodNominator.AddNominatedPod()出现在recordSchedulingFailure()函数中的原因。代码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/scheduler.go#L328>

```go
func (sched *Scheduler) recordSchedulingFailure(fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatedNode string) {
    // sched.Error是Pod调度出错后的处理，即便是抢占成功并提名，依然是资源不满足错误，因为等待被抢占Pod退出
    // 这个函数会调用PriorityQueue.AddUnschedulableIfNoPresent()函数将Pod放入不可调度子队列
    sched.Error(podInfo, err)
 
    // Update the scheduling queue with the nominated pod information. Without
    // this, there would be a race condition between the next scheduling cycle
    // and the time the scheduler receives a Pod Update for the nominated pod.
    // Here we check for nil only for tests.
    if sched.SchedulingQueue != nil {
        sched.SchedulingQueue.AddNominatedPod(podInfo.Pod, nominatedNode)
    }
 
    ......
}
```

而在调度器中调用recordSchedulingFailure()函数传入有效Node名字(非"")的地方只有<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/scheduler.go#L500>，表示抢占成功，其他的地方都是传入空的字符串，表示调度失败。本文不对调度器做过多分析，因为会有专门的文章分析，感兴趣的同学可以自行分析源码，当然也可以等待笔者的相关文章发布。笔者此处简单描述一下调度器抢占的原理：当没有任何一个Node满足Pod的需求的时候，调度器开始抢占Pod优先级低的Pod，如果抢占成功则提名被抢占Pod所在的Node，然后再将Pod放入不可调度队列，等待被抢占Pod退出。

当然，放入不可调度队列的Pod过一段时间还会被重新放入activeQ，此时如果有满足Pod需求的Node，kube-scheduler会将Pod调度到该Node上，然后清除Pod提名的Node。代码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/scheduler.go#L384>

```go
// 还记得《kube-scheduler的Cache解析》对于assume的解释么？如果忘记了请复习一遍
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
    // 设置Pod调度的Node
    assumed.Spec.NodeName = host
    // Cache中假定Pod已经调度到了Node上
    if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
        klog.Errorf("scheduler cache AssumePod failed: %v", err)
        return err
    }
    // 已经假定Pod调度到了Node上，应该移除以前提名的Node了(如果有提名的Node)
    if sched.SchedulingQueue != nil {
        sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
    }
 
    return nil
}
```

Pod通过PodNominator提名Node就可以么？下一轮调度新的Pod如何确保不会被重复抢占（Pod1抢占了Pod0等待的过程中，Pod2优先级不高于Pod1但高于Pod0的情况下不抢占Pod0）？这就是PodNominator.NominatedPodsForNode()的作用了。调度器在遍历Node时都会把这部分资源累加到Node上，这样就避免重复抢占的问题，这也是kube-scheduler里面比较常见的一个名词"Reserve"（预留）。因为这部分代码相对比较复杂，笔者此处不做注释。

以上就是PodNominator在kube-scheduler中的应用，如果感觉知识点有点零散不系统，那么请看下文的总结。

# 总结

1. PodNominator是kube-scheduler为了实现抢占调度定义的一种“提名”管理器，他记录了所有抢占成功但需要等被抢占Pod退出的Pod；
2. 通过PodNominator.NominatedPodsForNode()，kube-scheduler获取提名到指定Node上的所有Pod，在计算调度的时候这部分Pod申请的资源视为Node预留给抢占Pod的资源；
3. 当没有Node满足Pod的需求时，kube-scheduler开始执行抢占调度，如果抢占成功则利用PodNominator提名被抢占Pod所在的Node为Pod运行的Node；
4. 提名成功不代表已调度，所以此时Pod仍然是不可调度状态，放在调度队列的unschedulableQ子队列中；
5. 如果Pod从unschedulableQ迁移到activeQ，并且正好有Node满足Pod的需求，则Pod被调度到该Node上，并且删除以前提名的Node；
6. 此时再回来品一下“提名”，就是调度器预先在Node上为Pod预留一部分当前正在被别的Pod占用资源。此时再来看本文开始提出的问题，PodNominator有什么作用？我想笔者应该不用再做多余的解释了。
7. 最后，还有一个问题，为什么将PodNominator放在调度队列中实现？如果是笔者也会这么做，笔者的原因很简单：Pod虽然提名了Node，但是依然是未调度状态，所以放在调度队列中实现最合适。至于作者是不是这么考虑的就不知道了。
