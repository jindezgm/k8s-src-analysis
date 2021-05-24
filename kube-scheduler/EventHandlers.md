<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-24 21:05:32
 * @Description: 
-->

# 前言

笔者已经在[调度队列](./SchedulingQueue.md)、[调度框架](./Framework.md)、[调度插件](./Plugin.md)、[调度缓存](./Cache.md)以及[调度扩展器](./Extender.md)等文章中解析了kube-scheduler基本的调度原理与实现，但是有没有发现几个问题？

1. 需要调度的Pod是怎么进入[调度队列](./SchedulingQueue.md)的？
2. 调度完成的Pod是如何被更新到[调度缓存](./Cache.md)的？
3. Node的上线、下线等状态更新是如何更新到[调度缓存](./Cache.md)的？
4. 调度插件依赖的各种资源(比如PV)是如何影响调度的？

只有解答了以上问题后，才能把kube-scheduler各个功能模块串接起来，形成闭环。本文的目标就是解析kube-scheduler是如何感知调度依赖的各种资源的状态变化并做出相应的处理，看到这里相信有很多读者可以意识到client-go的SharedIndexInformer了。因为无论是Pod、Node还是PV，都是Kubernetes的一种API对象，而client-go为每一种API对象都提供了相应的SharedIndexInformer，对于使用者来说了只需要做相应的事件处理即可。kube-scheduler也是一样的，因为它也是apiserver的一个客户端，而Kubernetes中所有的API对象都是通过apiserver"广播"给所有的客户端并能够保持客户端之间状态的最终一致性。

说了这么多，简单一句话就是kube-scheduler需要**处理调度依赖对象**的事件(添加、更新、删除)，本文引用源码为kubernetes的release-1.21分支。

# 事件处理

## 注册

笔者在解析SharedIndexInformer的文章中已经提到，SharedIndexInformer的使用者只需要事件时间处理函数即可，现在我们来看看kube-scheduler总共注册了哪些对象事件处理函数，在后续的章节中笔者会一一解析每个事件的处理函数。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L358> 

```go
// addAllEventHandlers()用来注册kube-scheduler所有的事件处理函数，其中sched是调度器对象。
// 而informerFactory是kube-scheduler所有模块共享使用的SharedIndexInformer工厂。
func addAllEventHandlers(
	sched *Scheduler,
	informerFactory informers.SharedInformerFactory,
) {
	// 注册已调度Pod事件处理函数，可能有读者会问你怎么知道是已调度的Pod？这就要从过滤条件进行区分了。
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			// 事件过滤函数
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				// 如果对象是Pod，只有已调度的Pod才会通过过滤条件，assignedPod()函数判断len(pod.Spec.NodeName) != 0。
				// 所以此处断定是为已调度的Pod注册事件处理函数
				case *v1.Pod:
					return assignedPod(t)
				// 与上一个case相同，只是Pod处于已删除状态而已
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return assignedPod(pod)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				// 不是Pod类型，自然也就不可能通过过滤条件，当然这种可能性应该不大
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			// 注册已调度Pod事件处理函数，阅读过笔者关于调度缓存的文章的读者对此应该不陌生了吧。
			// 至此，结合调度框架相关的知识，笔者总结如下：
			// 1. 调度框架异步调用绑定插件将绑定子对象写入apiserver；
			// 2. SharedIndexInformer感知到Pod的状态更新，因为Pod.Spec.NodeName被设置，所以会通过过滤条件；
			// 3. kube-scheduler根据Pod的事件进行处理，进而更新Cache的状态；
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)
	// 注册未调度Pod事件处理函数，同样的道理，根据Pod的状态过滤未调度的Pod
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				// 对象类型为Pod
				case *v1.Pod:
					// 不仅Pod.Spec.NodeName没有被设置，同时Pod.Spec.SchedulerName指定的调度器是存在的。
					// 那么问题来了，Pod.Spec.SchedulerName指定的调度器不存在怎么办？无非是进不了调度队列，状态长期处于Pending而已。
					return !assignedPod(t) && responsibleForPod(t, sched.Profiles)
				// 同上，只是Pod处于已删除状态
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return !assignedPod(pod) && responsibleForPod(pod, sched.Profiles)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				// 非Pod类型不处理
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			// 注册未调度Pod事件处理函数，阅读过笔者关于调度队列的文章的读者对此应该不陌生了吧。
			// 无论是用kubectl创建的Pod还是各种controller创建的Pod，都是先在apiserver中创建对象，再通过apiserver通知到kube-scheduler
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)

	// 注册Node事件处理函数，此处了解调度缓存的的读者应该不陌生了吧
	informerFactory.Core().V1().Nodes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.addNodeToCache,
			UpdateFunc: sched.updateNodeInCache,
			DeleteFunc: sched.deleteNodeFromCache,
		},
	)

	// 注册CSINode事件处理函数，至于什么是CSINode，笔者在解析相应的处理函数的时候再做说明。
	informerFactory.Storage().V1().CSINodes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.onCSINodeAdd,
			UpdateFunc: sched.onCSINodeUpdate,
		},
	)

	// 注册PV事件处理函数，熟悉调度插件的读者应该不陌生，因为部分调度插件依赖PV，比如Pod需要挂载块设备
	informerFactory.Core().V1().PersistentVolumes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.onPvAdd,
			UpdateFunc: sched.onPvUpdate,
		},
	)

	// 注册PVC事件处理函数，没有PVC只有PV也是没用的，所以不用解释了
	informerFactory.Core().V1().PersistentVolumeClaims().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.onPvcAdd,
			UpdateFunc: sched.onPvcUpdate,
		},
	)

	// 注册Service事件处理函数，熟悉调度插件的读者应该不陌生，因为部分调度插件依赖Service，比如需要考虑服务亲和性，
	informerFactory.Core().V1().Services().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.onServiceAdd,
			UpdateFunc: sched.onServiceUpdate,
			DeleteFunc: sched.onServiceDelete,
		},
	)

	// 注册StorageClasse事件处理函数，与PV和PVC同
	informerFactory.Storage().V1().StorageClasses().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc: sched.onStorageClassAdd,
		},
	)
}
```

本文后续章节将逐一解析每个事件处理函数。

## 已调度Pod

先来看看已调度Pod事件处理函数，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L215> 

```go
// addPodToCache()是kube-scheduler处理已调度Pod的Added事件的函数。
func (sched *Scheduler) addPodToCache(obj interface{}) {
	// 以下这段内容没啥营养，不多解释
	pod, ok := obj.(*v1.Pod)
	if !ok {
		klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", obj)
		return
	}
	klog.V(3).InfoS("Add event for scheduled pod", "pod", klog.KObj(pod))

	// 函数的核心应该就是调用调度缓存(Cache)的接口将Pod添加到Cache中。
	if err := sched.SchedulerCache.AddPod(pod); err != nil {
		klog.ErrorS(err, "Scheduler cache AddPod failed", "pod", klog.KObj(pod))
	}

	// 这里有点意思了，已调度的Pod还要添加到调度缓存？还记得SchedulingQueue.AssignedPodAdded()这个函数的作用么？
	// 如果忘记了建议复读笔者解析调度队列的文章，其实这段代码的用意就是通知调度队列收到了已调度(Assigned指的是已分配Node)pod添加的消息，
	// 那么因为依赖该Pod被放入不可调度自子队列的Pod可以考虑进入active或者退避子队列了。
	// 所以此处并不是向调度队列添加Pod，而是触发调度队列更新可能依赖于此Pod的其他Pod的调度状态。
	sched.SchedulingQueue.AssignedPodAdded(pod)
}

// updatePodInCache()是kube-scheduler处理已调度Pod的Updated事件的函数。
func (sched *Scheduler) updatePodInCache(oldObj, newObj interface{}) {
	// 以下这段内容没啥营养，不多解释
	oldPod, ok := oldObj.(*v1.Pod)
	if !ok {
		klog.ErrorS(nil, "Cannot convert oldObj to *v1.Pod", "oldObj", oldObj)
		return
	}
	newPod, ok := newObj.(*v1.Pod)
	if !ok {
		klog.ErrorS(nil, "Cannot convert newObj to *v1.Pod", "newObj", newObj)
		return
	}

	// 能够进入这个函数说明新、旧Pod的名字和命名空间是相同的，而UID不同只能说明Pod经历了删除再添加的过程。
	// Pod删除事件和紧随其后的Pod添加事件可以合并为Pod更新事件，在这种情况下，应该使旧的Pod无效，然后添加新的Pod。
	if oldPod.UID != newPod.UID {
		sched.deletePodFromCache(oldObj)
		sched.addPodToCache(newObj)
		return
	}

	// 更新调度缓存(Cache)中的Pod。
	if err := sched.SchedulerCache.UpdatePod(oldPod, newPod); err != nil {
		klog.ErrorS(err, "Scheduler cache UpdatePod failed", "oldPod", klog.KObj(oldPod), "newPod", klog.KObj(newPod))
	}

	// 在addPodToCache()已经解释过，此处不赘述
	sched.SchedulingQueue.AssignedPodUpdated(newPod)
}

// deletePodFromCache()是kube-scheduler处理已调度Pod的Deleted事件的函数。
func (sched *Scheduler) deletePodFromCache(obj interface{}) {
	// 将interface{}转换为Pod
	var pod *v1.Pod
	switch t := obj.(type) {
	case *v1.Pod:
		pod = t
	case cache.DeletedFinalStateUnknown:
		var ok bool
		pod, ok = t.Obj.(*v1.Pod)
		if !ok {
			klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", t.Obj)
			return
		}
	default:
		klog.ErrorS(nil, "Cannot convert to *v1.Pod", "obj", t)
		return
	}
	klog.V(3).InfoS("Delete event for scheduled pod", "pod", klog.KObj(pod))
	// 将Pod从调度缓存(Cache)中删除
	if err := sched.SchedulerCache.RemovePod(pod); err != nil {
		klog.ErrorS(err, "Scheduler cache RemovePod failed", "pod", klog.KObj(pod))
	}

	// 与Pod的Added和Updated事件的处理方法类似，前两个都是更新依赖该Pod的Pod调度状态。
	// 对于Deleted事件则是更新所有不可调度的Pod的调度状态，笔者猜测应该暂时没有办法确定由于Pod的删除会影响哪些Pod调，所以采用如此简单粗暴的方法。
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.AssignedPodDelete)
}
```

## 未调度Pod

再来看看未调度Pod的事件处理函数，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L162> 

```go
// addPodToSchedulingQueue()是kube-scheduler处理未调度Pod的Added事件的函数。
// 比如通过kubectl创建的Pod，controller创建的Pod等等
func (sched *Scheduler) addPodToSchedulingQueue(obj interface{}) {
	// interface{}类型转Pod类型
	pod := obj.(*v1.Pod)
	klog.V(3).InfoS("Add event for unscheduled pod", "pod", klog.KObj(pod))
	// 函数实现很简单，直接添加到调度队列中
	if err := sched.SchedulingQueue.Add(pod); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
	}
}

// updatePodInSchedulingQueue()是kube-scheduler处理未调度Pod的Updated事件的函数。
// 还处在调度队列中Pod被更新，比如需更新资源需求、亲和性、标签等
func (sched *Scheduler) updatePodInSchedulingQueue(oldObj, newObj interface{}) {
	oldPod, newPod := oldObj.(*v1.Pod), newObj.(*v1.Pod)
	// 如果对象版本号没有更新则忽略，重复的Pod可能导致意外行为。那么问题来了:
	// 1. 为什么会有相同版本的对象的更新事件？不是只有对象更新的时候才会触发更新事件么？一旦对象更新版本肯定会更新。
	// 2. 即便版本号不变，说明对象也没有变化，会带来什么意外的行为？
	// 如果SharedIndexInformer设置了ResyncPeriod，那么就会周期性的重新同步对象，就会触发此类情况。
	// 至于会导致什么意外行为，其实没有人能够确定，只是这样做会避免不必要的处理，同时也就避免了出现问题的可能。
	if oldPod.ResourceVersion == newPod.ResourceVersion {
		return
	}
	// 其实在skipPodUpdate()函数中也能实现上面的判断，只是处理方法相对比较复杂。
	// skipPodUpdate()会忽略掉与调度相关字段没有更新的Pod，因为Pod更新有很多可能，不是每个字段更新都会影响调调度。
	// 下面有skipPodUpdate()函数的注释，会详细解析哪些Pod字段更新才会更新调度队列。
	if sched.skipPodUpdate(newPod) {
		return
	}
	// 更新调度队列中的Pod
	if err := sched.SchedulingQueue.Update(oldPod, newPod); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to update %T: %v", newObj, err))
	}
}

// deletePodFromSchedulingQueue()是kube-scheduler处理未调度Pod的Deleted事件的函数。
// 还处于调度队列中的Pod被删除，这种情况很常见，比如利用kubectl创建的Pod一直处于Pending状态，
// 人工分析发现某些字段有问题，然后再用kubectl删除掉该Pod，至少笔者经常这么干...
func (sched *Scheduler) deletePodFromSchedulingQueue(obj interface{}) {
	// interface{}转为Pod类型
	var pod *v1.Pod
	switch t := obj.(type) {
	case *v1.Pod:
		pod = obj.(*v1.Pod)
	case cache.DeletedFinalStateUnknown:
		var ok bool
		pod, ok = t.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
			return
		}
	default:
		utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
		return
	}
	klog.V(3).InfoS("Delete event for unscheduled pod", "pod", klog.KObj(pod))
	// 将Pod从调度队列中删除
	if err := sched.SchedulingQueue.Delete(pod); err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to dequeue %T: %v", obj, err))
	}
	// 根据Pod.Spec.SchedulerName获取调度框架(调度器)
	fwk, err := sched.frameworkForPod(pod)
	if err != nil {
		klog.ErrorS(err, "Unable to get profile", "pod", klog.KObj(pod))
		return
	}
	// 弹出等待的Pod，了解调度插件的读者应该不陌生，PermitPlugin插件会让Pod等待，所以如果Pod处于等待状态则从等待类表中删除。
	// 因为等待Pod是单独管理的(WaitingPod)，并且Pod没有字段指定其是否为等待状态，所以这是"盲目"删除。
	fwk.RejectWaitingPod(pod.UID)
}

// skipPodUpdate()检查是否应忽略Pod更新，同时满足以下情况函数会返回true：
// 1. 该Pod已经被假定调度，笔者在解析调度缓存(Cache)的文章中引入了假定调度，即已分配Node但还没有绑定完成的Pod;
// 2. Pod仅更新了ResourceVersion，Spec.NodeName，Annotations，ManagedFields，Finalizer, Condition;
// 总结一句话就是，已假定调度的Pod但是只更新某些字段，这些字段不会引起该Pod重新调度，则会忽略。
func (sched *Scheduler) skipPodUpdate(pod *v1.Pod) bool {
	// 如果是未假定调度的Pod不需要忽略
	isAssumed, err := sched.SchedulerCache.IsAssumedPod(pod)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("failed to check whether pod %s/%s is assumed: %v", pod.Namespace, pod.Name, err))
		return false
	}
	if !isAssumed {
		return false
	}

	// 获取假定调度Pod的状态，assumedPod保存的是更新前的值
	assumedPod, err := sched.SchedulerCache.GetPod(pod)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("failed to get assumed pod %s/%s from cache: %v", pod.Namespace, pod.Name, err))
		return false
	}

	// 定义一个函数，该函数屏蔽掉Pod不需要关心的字段，这样就可以利用reflect.DeepEqual()函数比较两个对象是否有更新。
	f := func(pod *v1.Pod) *v1.Pod {
		p := pod.DeepCopy()
		// 屏蔽版本，因为对象更新都会附带版本的更新(Resync的情况已经在前面的代码处理过了）。
		// 简单一句话：Pod对象更新版本肯定会变化，但是版本变化不一定影响调度决策。
		p.ResourceVersion = ""
		// 假定调度的Pod会设置Spec.NodeName字段，但是更新Pod可能没有设置这个字段，所以屏蔽掉。
		// 因为Pod更新和绑定是两个并行的行为，如果更新事件在绑定之前发生，此时忽略掉Pod.Spec.NodeName的差异。
		p.Spec.NodeName = ""
		// 必须屏蔽注解，具体原因参看https://github.com/kubernetes/kubernetes/issues/52914。
		// 说的简单点就是调度过程中有些插件/调度扩展程序可能会修改注解字段。
		p.Annotations = nil
		// 同上，当使用ServerSideApply修改注释时，ManagedFields也可能会更改。
		// 关于ServerSideApply参看https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/
		p.ManagedFields = nil
		// controller可能会更改以下字段，但它们不会影响调度决策，所以屏蔽掉
		p.Finalizers = nil
		p.Status.Conditions = nil
		return p
	}
	// 利用relect.DeepEqual()判断更新后的Pod是否有变化，忽略掉更新没有任何变化的Pod
	assumedPodCopy, podCopy := f(assumedPod), f(pod)
	if !reflect.DeepEqual(assumedPodCopy, podCopy) {
		return false
	}
	klog.V(3).InfoS("Pod update ignored because changes won't affect scheduling", "pod", klog.KObj(pod))
	return true
}
```

## Node

已经知道kube-scheduler如何处理Pod事件，接下来看看如何处理Node的事件，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L90> 

```go
// addNodeToCache()是kube-scheduler处理Node的Added事件的函数。比如，集群管理员向集群添加了Node。
func (sched *Scheduler) addNodeToCache(obj interface{}) {
	// 将interface{}转为Node
	node, ok := obj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert to *v1.Node", "obj", obj)
		return
	}

	// 将Node添加到调度缓存(Cache)
	if err := sched.SchedulerCache.AddNode(node); err != nil {
		klog.ErrorS(err, "Scheduler cache AddNode failed")
	}

	// 因为有新的Node添加，所有不可调度的Pod都有可能可以被调度了，这种实现还是相对简单粗暴
	klog.V(3).InfoS("Add event for node", "node", klog.KObj(node))
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.NodeAdd)
}

// updateNodeInCache()是kube-scheduler处理Node的Updated事件的函数。比如，集群管理员为Node打了标签。
func (sched *Scheduler) updateNodeInCache(oldObj, newObj interface{}) {
	// 将interface{}转为Node
	oldNode, ok := oldObj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert oldObj to *v1.Node", "oldObj", oldObj)
		return
	}
	newNode, ok := newObj.(*v1.Node)
	if !ok {
		klog.ErrorS(nil, "Cannot convert newObj to *v1.Node", "newObj", newObj)
		return
	}

	// 更新调度缓存(Cache)中的Node
	if err := sched.SchedulerCache.UpdateNode(oldNode, newNode); err != nil {
		klog.ErrorS(err, "Scheduler cache UpdateNode failed")
	}

	// 仅在Node变得更可调度时才重新调度不可调度的Pod，什么叫做更可调度？比如可分配资源增加、以前不可调度现在可调度等。
	// 下面有nodeSchedulingPropertiesChange()的注释。
	if event := nodeSchedulingPropertiesChange(newNode, oldNode); event != "" {
		sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(event)
	}
}

// deleteNodeFromCache()是kube-scheduler处理Node的Deleted事件的函数。比如，集群管理员将Node从集群中删除。
func (sched *Scheduler) deleteNodeFromCache(obj interface{}) {
	// 将interface{}转为Node
	var node *v1.Node
	switch t := obj.(type) {
	case *v1.Node:
		node = t
	case cache.DeletedFinalStateUnknown:
		var ok bool
		node, ok = t.Obj.(*v1.Node)
		if !ok {
			klog.ErrorS(nil, "Cannot convert to *v1.Node", "obj", t.Obj)
			return
		}
	default:
		klog.ErrorS(nil, "Cannot convert to *v1.Node", "obj", t)
		return
	}
	klog.V(3).InfoS("Delete event for node", "node", klog.KObj(node))
	// 将Node从调度缓存(Cache)中删除
	if err := sched.SchedulerCache.RemoveNode(node); err != nil {
		klog.ErrorS(err, "Scheduler cache RemoveNode failed")
	}
}

// nodeSchedulingPropertiesChange()并不是判断Node是否更可调度，而是判断可调度属性发生变化。
// 如果Node向不可调度变化其实不应该触发重新调度不可调度Pod，但是也不会引起什么问题，无非多一点不必要的计算而已。
// 但是这样的实现比较简单，不容易出错，毕竟什么是更可调度不是一两句话可以说清楚的，而判断是否变化是很容易的。
func nodeSchedulingPropertiesChange(newNode *v1.Node, oldNode *v1.Node) string {
	// Node.Spec.Unschedulable发生变化
	if nodeSpecUnschedulableChanged(newNode, oldNode) {
		return queue.NodeSpecUnschedulableChange
	}
	// Node.Status.Allocatable发生变化
	if nodeAllocatableChanged(newNode, oldNode) {
		return queue.NodeAllocatableChange
	}
	// Node.Labels发生变化
	if nodeLabelsChanged(newNode, oldNode) {
		return queue.NodeLabelChange
	}
	// Node.Spec.Taints发生变化
	if nodeTaintsChanged(newNode, oldNode) {
		return queue.NodeTaintChange
	}
	// Node.Status.Conditions发生变化
	if nodeConditionsChanged(newNode, oldNode) {
		return queue.NodeConditionChange
	}

	return ""
}
```

## CSINode

是时候介绍什么是CSINode了，请阅读[CSINode官方文档](https://kubernetes-csi.github.io/docs/csi-node-object.html)，哈哈，是不是感觉被套路了，既然有官方文档，笔者就不献丑了。了解了CSINode，来看看kube-scheduler如何处理CSINode的事件，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L154> 

```go
// onCSINodeAdd()是kube-scheduler处理CSINode的Added事件的函数。
func (sched *Scheduler) onCSINodeAdd(obj interface{}) {
	// 重新调度所有不可调度的Pod
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.CSINodeAdd)
}

// onCSINodeUpdate()是kube-scheduler处理CSINode的Updated事件的函数。
func (sched *Scheduler) onCSINodeUpdate(oldObj, newObj interface{}) {
	// 重新调度所有不可调度的Pod
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.CSINodeUpdate)
}
```

有没有感觉被骗了，这根本不需要了解CSINode是啥。其实不然，在不了解CSINode的情况下应该很难理解为啥重新调度不可调度Pod。

## Service

能够触发重新调度不可调度Pod的事件不仅有CSINode，还有Service的事件，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L78>

```go
// 根据以下几个函数可知，Service的任何事件都会重新调度不可调度Pod
func (sched *Scheduler) onServiceAdd(obj interface{}) {
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.ServiceAdd)
}
func (sched *Scheduler) onServiceUpdate(oldObj interface{}, newObj interface{}) {
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.ServiceUpdate)
}
func (sched *Scheduler) onServiceDelete(obj interface{}) {
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.ServiceDelete)
}
```

## PV&PVC&StorageClass

你以为只有CSINode、Service会"无脑"重新调度不可调度Pod？其实PV、PVC和StorageClass也是如此，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/eventhandlers.go#L34>

```go
// onPvAdd()是kube-scheduler处理PV的Added事件的函数。
func (sched *Scheduler) onPvAdd(obj interface{}) {
	// Pod因为没有可用PV时将卡在无法调度的队列中。
	// 静态供应(static provisioning)创建的未绑定的PV和PV controller动态供应和绑定的过程中忽略的延迟绑定的StorageClass，不会触发事件再次调度pod
	// 因此，在这种情况下，当添加PV时需要将Pod移到active队列中。
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.PvAdd)
}

// onPvUpdate()是kube-scheduler处理PV的Updated事件的函数。
func (sched *Scheduler) onPvUpdate(old, new interface{}) {
	// kube-scheduler为Pod绑定卷时可能会失败，比如PV controller更新了PV导致假定调度的Pod绑定卷是冲突。
	// 然后kube-scheduler将Pod添加回无法调度的队列，在这种情况下，需要将Pod移至active队列中。
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.PvUpdate)
}

// onPvcAdd()是kube-scheduler处理PVC的Added事件的函数。
func (sched *Scheduler) onPvcAdd(obj interface{}) {
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.PvcAdd)
}

// onPvcUpdate()是kube-scheduler处理PVC的Updated事件的函数。
func (sched *Scheduler) onPvcUpdate(old, new interface{}) {
	sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.PvcUpdate)
}

// onStorageClassAdd()是kube-scheduler处理StorageClass的Added事件的函数。
func (sched *Scheduler) onStorageClassAdd(obj interface{}) {
	sc, ok := obj.(*storagev1.StorageClass)
	if !ok {
		klog.ErrorS(nil, "Cannot convert to *storagev1.StorageClass", "obj", obj)
		return
	}

	// 如果Pod具有未绑定immediate PVC(实在不知道怎么翻译好了...)，则过滤会(FilterPlugin)失败。
	// 如果这些PVC指定StorageClass，创建后期绑定的StorageClass对象将导致过滤通过，因此需要将Pod移至active队列。
	if sc.VolumeBindingMode != nil && *sc.VolumeBindingMode == storagev1.VolumeBindingWaitForFirstConsumer {
		sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.StorageClassAdd)
	}
}
```

# 总结

1. kube-scheduler处理了Pod、Node、CSINode、PV、PVC、Service以及StorageClass对象的事件，基于事件的类型(添加、删除、更新)执行对应的操作；
2. 未调度的Pod会被SchedulingQueue和WaitingPod管理，所以删除时需要从这两个对象中删除；
3. Pod、Node的事件处理会根据状态(已调度、未调度)和事件类型操作调度队列和调度缓存；
4. CSINode、PV、PVC、Service以及StorageClass的事件只是重新调度不可调度的Pod，因为这些事件可能会导致一些不可调度的Pod可调度；
