<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-28 07:47:31
 * @Description: Controller(Kubernetes)的ControllerExpectations源码解析
-->

# 前言

本文涉及的部分名词参看[名词解释](./README.md)，此处还要强调两个概念：Controller和Controllee，在本文其实二者是同一个意思，代表XxxController(比如DeploymentController)管理的Xxx(比如Deployment)对象。ControllerExpectations可以看做是XxxExpectations，只是各种Xxx的Expectations接口是相同的，就用ControllerExpectations统一实现了。

要想理解Kubernetes的Controller，需要对异步有足够的了解，否则XxxController的源码实现可能会让人看得非常懵逼。ControllerExpectations是XxxContrller中实现同步操作的Xxx的关键，它有点类似于sync.WaitGroup增加了非阻塞Wait()接口。以ReplicaSetController为例，根据ReplicaSet的声明创建N个Pod，此时用ControllerExpectations记录该ReplicaSet期望创建的Pod的数量为N(等同于sync.WaitGroup.Add(N))。ReplicaSetController监视(SharedIndexInformer的Pod的事件)到该ReplicaSet的一个Pod创建成功就会通知ControllerExpectations对期望创建的Pod数量减一(等同于sync.WaitGroup.Done())。直到达到期望创建的Pod数量归0(N个Pod全部创建完成)，ReplicaSetController才会根据ReplicaSet的状态做下一步操作，直到达到Xxx声明的状态为止。那么，从ReplicaSetController从创建N个Pod开始直到确认N个Pod创建完成，整个过程其实存在很多异步操作（例如SharedIndexInformer的各种事件），因为有ControllerExpectations的存在，让整个操作跟同步操作一样，这也是很多XxxController都有一个函数叫做syncXxx()的原因，它等同于同步处理一个Xxx对象。类似用异步操作实现成同步操作的方法很多，比如用sync.Cond。

当然，XxxController并不是只创建Xxx子对象，还会删除Xxx的子对象，所以ControllerExpectations中有两个计数，分别用于创建和删除子对象的期望数量。

本文引用源码为kubernetes的release-1.20分支。

# ControllerExpectations

## ControllerExpectationsInterface

ControllerExpectationsInterface是ControllerExpectations抽象接口，这个抽象接口仅仅为了单元测试使用的(即便有一些XxxController已经使用了这个接口)，所以读者简单了解一下就可以了。源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L144>。

```go
type ControllerExpectationsInterface interface {
    // 获取Controller的期望，ControlleeExpectations后文有介绍，此处的controllerKey就是Xxx的唯一键，即NS/Name
    GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error)
    // 判断Controller的期望是否达成
    SatisfiedExpectations(controllerKey string) bool
    // 删除Controller的期望
    DeleteExpectations(controllerKey string)
    // 设置Controller的期望，其中add是创建子对象的数量，del是删除子对象的数量
    SetExpectations(controllerKey string, add, del int) error
    // Controller期望创建adds个子对象
    ExpectCreations(controllerKey string, adds int) error
    // Controller期望删除dels个子对象
    ExpectDeletions(controllerKey string, dels int) error
    // 观测到Controller的一个子对象创建完成，此时Controller期望创建的子对象数量减一
    CreationObserved(controllerKey string)
    // 观测到Controller的一个子对象删除完成，此时Controller期望删除的子对象数量减一
    DeletionObserved(controllerKey string)
    // 提升Controller的期望，add和del分别是创建和删除子对象的增量
    RaiseExpectations(controllerKey string, add, del int)
    // 降低Controller的期望，add和del分别是创建和删除子对象的增量
    LowerExpectations(controllerKey string, add, del int)
}
```

## ControlleeExpectations

ControllerExpectations管理所有Xxx(比如ReplicaSet)对象的期望，ControlleeExpectations就是某一个Xxx对象的期望。ControllerExpectations可以简单理解为map\[string\]ControlleeExpectations结构，map的键是Xxx对象的唯一键(比如NS/Name)。源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L266>。

```go
type ControlleeExpectations struct {
    // 期望创建/删除子对象(比如Pod)的数量
    add       int64
    del       int64
    // Controller的唯一键
    key       string
    // 创建时间
    timestamp time.Time
}

// 增加期望值，参数是创建和删除的增量值
func (e *ControlleeExpectations) Add(add, del int64) {
    atomic.AddInt64(&e.add, add)
    atomic.AddInt64(&e.del, del)
}

// 期望是否达成
func (e *ControlleeExpectations) Fulfilled() bool {
    // 如何才算是期望达成？就是期望创建和删除的子对象是否归零，如果二者全部归零说明没有任何期望了，也就是期望达成了
    return atomic.LoadInt64(&e.add) <= 0 && atomic.LoadInt64(&e.del) <= 0
}

// 获取期望值，需要返回创建和删除子对象的期望值
func (e *ControlleeExpectations) GetExpectations() (int64, int64) {
    return atomic.LoadInt64(&e.add), atomic.LoadInt64(&e.del)
}

// 判断期望是否过期(超时)，这是一个比较有用的接口，一旦有什么异常造成期望长期无法达成，Controller就会一直没有进展。
// 利用超时机制可以解决此类异常(超时真是分布式系统中非常好用的方法)，期望过期等于期望达成(无非可能是一个0值期望)，XxxController会继续根据Xxx当前的状态进一步调整以达到Xxx声明的状态。
// 这种异常很常见？笔者举个不恰当的例子，只是为了方便理解: 比如删除一个Pod，Pod僵死一直不退出，那么期望删除一个Pod就会一直无法达成，Xxx的状态就一直没有进展。
func (exp *ControlleeExpectations) isExpired() bool {
    // 从创建期望到现在超过ExpectationsTimeout(5分钟)则认为过期
    return clock.RealClock{}.Since(exp.timestamp) > ExpectationsTimeout
}
```

## ControllerExpectations

XxxController利用ControllerExpectations管理了所有Xxx对象的期望，源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L158>。

```go
// ControllerExpectations实现了ControllerExpectationsInterface。
type ControllerExpectations struct {
    // cache包是client-go的cache，基本等同于map，不了解的读者可以阅读笔者关于client-go的Cache的文章
    cache.Store
}

// SatisfiedExpectations实现了ControllerExpectationsInterface.SatisfiedExpectations()接口
func (r *ControllerExpectations) SatisfiedExpectations(controllerKey string) bool {
    // 获取Controller的期望
    if exp, exists, err := r.GetExpectations(controllerKey); exists {
        if exp.Fulfilled() {
            // 期望已达成
            klog.V(4).Infof("Controller expectations fulfilled %#v", exp)
            return true
        } else if exp.isExpired() {
            // 期望已过期，返回true，这样XxxController可以根据Xxx最新的状态进行调整
            klog.V(4).Infof("Controller expectations expired %#v", exp)
            return true
        } else {
            // 期望未达成且未过期，继续等待期望达成
            klog.V(4).Infof("Controller still waiting on expectations %#v", exp)
            return false
        }
    } else if err != nil {
        // 获取期望错误
        klog.V(2).Infof("Error encountered while checking expectations %#v, forcing sync", err)
    } else {
        // 期望不存在
        klog.V(4).Infof("Controller %v either never recorded expectations, or the ttl expired.", controllerKey)
    }
    // 获取期望错误或者期望不存在，对于Controller来说等同于期望达成，否则Controller将不会有任何进展。
    // 什么是期望达成？就是Xxx上一次的操作已经完成，告知XxxController可以对Xxx执行下一步操作了，关键在于可以开始下一步工作了。
    // 所以在必要(比如出现异常)的时候，也可以看做是期望达成，可能此时期望值是0。
    return true
}

// ControllerExpectations其他接口的实现读者自己看就行了，非常简单，都是利用ControlleeExpectations实现的。
```

# UIDTrackingControllerExpectations

UIDTrackingControllerExpectations从名字上看继承了ControllerExpectations，同时还有跟踪UID的能力，此处的UID指导就是API对象的UID。所以期望已经不仅仅是数量了，还有具体是哪些对象，这可以得知期望是有状态的，因为他比ControllerExpectations多了指定的UID。换句话说，无论是创建还是删除子对象，必须是指定的那些对象，否则期望不会达成。所以UIDTrackingControllerExpectations就用在StatefulSetController中。<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L319>。

```go
type UIDTrackingControllerExpectations struct {
    // 继承了ControllerExpectationsInterface
    ControllerExpectationsInterface
    // 用cache.Store加锁管理跟踪的UID，需要注意的是跟踪的UID只用于期望删除的Pod，期望创建的Pod不需要跟踪UID。
    uidStoreLock sync.Mutex
    uidStore cache.Store
}

// 获取Controller跟踪的UID，是一个字符串集合
func (u *UIDTrackingControllerExpectations) GetUIDs(controllerKey string) sets.String {
    // 函数没有上锁，所以调用此函数的地方需要加锁保护
    if uid, exists, err := u.uidStore.GetByKey(controllerKey); err == nil && exists {
        // UIDSet是一个结构体，包括string.Set(String)字段和一个key，读者可以自己看一下
        return uid.(*UIDSet).String
    }
    return nil
}

// ExpectDeletions覆盖了ControllerExpectationsInterface.ExpectDeletions()，参数从dels整数变成了字符串slice。
// 这一点可以看出跟踪UID是删除子对象的UID。
func (u *UIDTrackingControllerExpectations) ExpectDeletions(rcKey string, deletedKeys []string) error {
    // 字符串slice转set
    expectedUIDs := sets.NewString()
    for _, k := range deletedKeys {
        expectedUIDs.Insert(k)
    }
    klog.V(4).Infof("Controller %v waiting on deletions for: %+v", rcKey, deletedKeys)
    u.uidStoreLock.Lock()
    defer u.uidStoreLock.Unlock()

    // 如果已存在并跟踪了一些UID，就写错误日志，也就是说理论上不应该发生这种情况
    if existing := u.GetUIDs(rcKey); existing != nil && existing.Len() != 0 {
        klog.Errorf("Clobbering existing delete keys: %+v", existing)
    }
    // 记录新的UID集合
    if err := u.uidStore.Add(&UIDSet{expectedUIDs, rcKey}); err != nil {
        return err
    }
    // 设置期望删除子对象的数量
    return u.ControllerExpectationsInterface.ExpectDeletions(rcKey, expectedUIDs.Len())
}

// DeletionObserved()覆盖了ControllerExpectationsInterface.DeletionObserved()，增加了已删除子对象的UID(deleteKey)。
func (u *UIDTrackingControllerExpectations) DeletionObserved(rcKey, deleteKey string) {
    u.uidStoreLock.Lock()
    defer u.uidStoreLock.Unlock()

    // deleteKey必须是跟踪的UID，否则不会影响期望值
    uids := u.GetUIDs(rcKey)
    if uids != nil && uids.Has(deleteKey) {
        klog.V(4).Infof("Controller %v received delete for pod %v", rcKey, deleteKey)
        u.ControllerExpectationsInterface.DeletionObserved(rcKey)
        uids.Delete(deleteKey)
    }
}

// DeleteExpectations()覆盖了ControllerExpectationsInterface.DeleteExpectations()。
func (u *UIDTrackingControllerExpectations) DeleteExpectations(rcKey string) {
    u.uidStoreLock.Lock()
    defer u.uidStoreLock.Unlock()

    // 删除Controller的期望，同时删除跟踪的UID(如果存在的话)
    u.ControllerExpectationsInterface.DeleteExpectations(rcKey)
    if uidExp, exists, err := u.uidStore.GetByKey(rcKey); err == nil && exists {
        if err := u.uidStore.Delete(uidExp); err != nil {
            klog.V(2).Infof("Error deleting uid expectations for controller %v: %v", rcKey, err)
        }
    }
}
```
