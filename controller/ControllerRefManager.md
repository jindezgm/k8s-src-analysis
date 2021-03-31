<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-28 07:46:34
 * @Description: ControllerRefManager源码解析
-->

# 前言

[名词解释](./README.md)中已经解释了什么是Controller，那么ControllerRef就是对Controller的引用。在单进程编程中引用一般等同于指针(指针是对象的唯一标识)，但是分布式系统中引用一般是对象的唯一键，这个唯一键可能是多维度的（以K8S为例：NS、Name、Kind等）。ControllerRefManager从名词上理解就是管理对象的Controller引用，而本文的Controller指的是对象的父对象（Owner），所以ControllerRefManager用来管理对象的Onwer引用的。比如某个Pod的Owner是ReplicaSet，同时该ReplicaSet的Owner是一个Deployment，所以ControllerRefManager用来管理Pod的父对象ReplicaSet和ReplicaSet的父对象Deployment。

本文应用源码为kubernetes的release-1.21分支。

# OwnerReference

现在我们来看看Kubernetes里对于Owner引用的定义，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/apis/meta/v1/types.go#L306>

```go
type OwnerReference struct {
    // 拥有者的API version，例如ReplicaSet对应的APIVersion可以是apps/v1
    APIVersion string `json:"apiVersion" protobuf:"bytes,5,opt,name=apiVersion"`
    // 拥有者的API Kind，复用上面的例子，Kind就是ReplicaSet
    Kind string `json:"kind" protobuf:"bytes,1,opt,name=kind"`
    // 拥有者的名字，复用上面的例子就是ReplicaSet的名字。
    // 有没有发现没有NS？这是为什么？不是NS/Name才是唯一键么？道理很简单，父对象和子对象在同一个NS下，所以没有NS。
    Name string `json:"name" protobuf:"bytes,3,opt,name=name"`
    // 拥有者的UID，复用上面的例子就是ReplicaSet的UID，对象删除又创建会造成NS/Name相同但UID不同
    UID types.UID `json:"uid" protobuf:"bytes,4,opt,name=uid,casttype=k8s.io/apimachinery/pkg/types.UID"`
    // 如果为true，表示拥有者是Controller类型，比如Deployment、ReplicaSet、Job等
    Controller *bool `json:"controller,omitempty" protobuf:"varint,6,opt,name=controller"`
    // 如果为true，并且拥有者通过foreground(例如kubectl delete xxx)删除时，只有该引用被删除时拥有者对象才能删除。
    BlockOwnerDeletion *bool `json:"blockOwnerDeletion,omitempty" protobuf:"varint,7,opt,name=blockOwnerDeletion"`
}
```

# ControllerRefManager

## BaseControllerRefManager

BaseControllerRefManager是XxxControllerRefManager的基类，此处Xxx就是需要设置拥有者的对象类型，比如Pod或者ReplicaSet(后文会有介绍)。源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_ref_manager.go#L35>。

```go
type BaseControllerRefManager struct {
    // Controller(Owner)的meta
    Controller metav1.Object
    // Controller匹配子对象的Selector，很好理解，很为我们经常在yaml中写Label和Selector。
    Selector   labels.Selector

    // 判断是否可以接纳子对象的函数，函数由外部传入，并且只会调用一次，后面介绍PodControllerRefManager的时候会详细说明，此处先忽略。
    // 什么是接纳？这要从没有ControllerRef的对象说起，这些对象对于Controller来说是‘孤儿’，只要是XxxController创建的子对象都会设置ControllerRef。
    // 如果通过子对象的标签匹配选择器，就可以考虑是否接纳这个子对象，即为这个子对象设置ControllerRef。当然，接纳孤儿没有那么简单，后面会详细说明。
    // 笔者将adopt翻译为‘接纳’，寓意为本身不属于Controller的子对象，经过各种匹配后才会被接纳，即获得子对象的拥有权。
    canAdoptErr  error
    canAdoptOnce sync.Once
    CanAdoptFunc func() error
}

// CanAdopt()将canAdoptErr、canAdoptOnce、CanAdoptFunc组合成判断能否可以接纳对象的函数。
// 为什么只调用一次，这个在具体使用的地方说明更加合适，此处只需要知道只调用一次即可。
// 至少知道一点，能否能够接纳与具体的对象无关，因为函数不传入任何参数，所以可以推测应该与Controller的状态有关。
func (m *BaseControllerRefManager) CanAdopt() error {
    m.canAdoptOnce.Do(func() {
        if m.CanAdoptFunc != nil {
            m.canAdoptErr = m.CanAdoptFunc()
        }
    })
    return m.canAdoptErr
}

// ClaimObject()尝试获取对象(obj)的拥有权，match是匹配函数，adopt和release是接纳和释放obj拥有权的函数。
// 1. 如果match()返回true，则接纳孤儿；
// 2. 如果match()返回false，则释放对象的拥有权；
// 那么问题来了，已经有选择器了，为什么还要传入匹配函数？匹配什么呢？这个笔者在调用该函数的地方再说明。
func (m *BaseControllerRefManager) ClaimObject(obj metav1.Object, match func(metav1.Object) bool, adopt, release func(metav1.Object) error) (bool, error) {
    // 感兴趣的读者可以阅读GetControllerOfNoCopy()函数源码，笔者在此简单描述一下这个函数的功能：
    // 遍历obj.OwnerReferences，找到第一个*obj.OwnerReferences[i].Controller == true的引用。
    // 此处也可以得出一个结论：子对象只能有一个Controller类型的拥有者，否则不会发现第一个ControllerRef就返回。
    controllerRef := metav1.GetControllerOfNoCopy(obj)
    // 如果controllerRef不为空，说明obj已经有一个Controller的父对象
    if controllerRef != nil {
        // Owner是自己么？
        if controllerRef.UID != m.Controller.GetUID() {
            // 对象的拥有者不是自己，返回失败，毕竟对象已经有ControllerRef。
            return false, nil
        }
        // 匹配对象
        if match(obj) {
            // 已经拥有对象并且匹配成功，返回成功
            return true, nil
        }
        // 拥有对象但是匹配失败，如果Controller正在删除则不用释放，因为Controller删除会释放对象拥有权
        if m.Controller.GetDeletionTimestamp() != nil {
            return false, nil
        }
        // 拥有对象但是匹配失败，释放对象拥有权。前面提到孤儿的时候并没有解释孤儿怎么产生，此处就是产生孤儿的一种情况。
        // 即Controller与子对象不匹配，此时将释放子对象的拥有权。
        if err := release(obj); err != nil {
            // Pod不存在，等同于释放成功
            if errors.IsNotFound(err) {
                return false, nil
            }
            // 释放失败
            return false, err
        }
        // 释放成功
        return false, nil
    }

    // 因为obj没有ControllerRef，所以它是一个孤儿对象，如果Controller正在被删除亦或是匹配失败，则无法接纳它
    if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
        return false, nil
    }
    // 不能接纳一个正在删除的孤儿对象
    if obj.GetDeletionTimestamp() != nil {
        return false, nil
    }
    // 匹配成功，尝试接纳孤儿
    if err := adopt(obj); err != nil {
        // 对象不存在，接纳失败，这并不是错误，因为对象已经被删除了
        if errors.IsNotFound(err) {
            return false, nil
        }
        // 接纳错误
        return false, err
    }
    // 接纳成功
    return true, nil
}

```

# PodControllerRefManager

PodControllerRefManager用于管理Pod的ControllerRef，也就是说PodControllerRefManager.ClaimObject()函数传入的对象都是Pod，源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_ref_manager.go#L123>。

```go
type PodControllerRefManager struct {
    // 继承了BaseControllerRefManager，因为BaseControllerRefManager用于管理通用对象的ControllerRef
    BaseControllerRefManager
    // ControllerRef的Group、Version、Kind三元组，比如apps、v1、ReplicaSet
    controllerKind schema.GroupVersionKind
    // PodControlInterface用于创建/删除/更新Pod，详情查看：https://github.com/jindezgm/k8s-src-analysis/blob/master/controller/PodControl.md
    podControl     PodControlInterface
}

// NewPodControllerRefManager()是PodControllerRefManager的构造函数，由kube-controller-manager调用。
func NewPodControllerRefManager(
    podControl PodControlInterface,
    controller metav1.Object,
    selector labels.Selector,
    controllerKind schema.GroupVersionKind,
    canAdopt func() error,
) *PodControllerRefManager {
    // PodControllerRefManager所有参数基本都是外部传入的...
    return &PodControllerRefManager{
        BaseControllerRefManager: BaseControllerRefManager{
            Controller:   controller,
            Selector:     selector,
            CanAdoptFunc: canAdopt,
        },
        controllerKind: controllerKind,
        podControl:     podControl,
    }
}

// ClaimPods()尝试获取一组Pod的拥有权，同时传入了一组过滤函数(想必是给BaseControllerRefManager.ClaimObject()的匹配函数使用的)。
func (m *PodControllerRefManager) ClaimPods(pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
    var claimed []*v1.Pod
    var errlist []error

    // 终于看到匹配函数了，Pod的匹配函数是标签匹配同时通过所有过滤器，标签选择器好理解，过滤器到底过滤了啥就得看XxxController的具体实现了。
    match := func(obj metav1.Object) bool {
        pod := obj.(*v1.Pod)
        if !m.Selector.Matches(labels.Set(pod.Labels)) {
            return false
        }
        for _, filter := range filters {
            if !filter(pod) {
                return false
            }
        }
        return true
    }
    // 接纳Pod和释放Pod拥有权的函数，下面有这两个函数的注释
    adopt := func(obj metav1.Object) error {
        return m.AdoptPod(obj.(*v1.Pod))
    }
    release := func(obj metav1.Object) error {
        return m.ReleasePod(obj.(*v1.Pod))
    }

    // 遍历Pod，逐个获得/释放Pod拥有权
    for _, pod := range pods {
        // 尝试获取Pod的拥有权
        ok, err := m.ClaimObject(pod, match, adopt, release)
        if err != nil {
            errlist = append(errlist, err)
            continue
        }
        if ok {
            claimed = append(claimed, pod)
        }
    }
    // 返回获得拥有权的Pod以及没有获得拥有权的错误
    return claimed, utilerrors.NewAggregate(errlist)
}

// AdoptPod()发送Patch更新请求到apieserver来获得Pod的拥有权，将ControllerRef合并到Pod.OwnerReferences.
func (m *PodControllerRefManager) AdoptPod(pod *v1.Pod) error {
    // 还记得CanAdopt()是一个只会真正调用一次的函数么？此处需要知道PodControllerRefManager.ClaimPods()传入的是多个Pod。
    // 也就是说这些Pod如果有多个Pod需要校验CanAdopt()，则第一个Pod决定了后面所有的Pod，即要么都能接纳，要么都不能接纳。
    // 这更加证明了能否接纳子对象看Controller的状态而不是子对象的状态。
    if err := m.CanAdopt(); err != nil {
        return fmt.Errorf("can't adopt Pod %v/%v (%v): %v", pod.Namespace, pod.Name, pod.UID, err)
    }
    
    // ownerRefControllerPatch()会生成ControllerRef的补丁，读者可以自己看一下，非常简单
    patchBytes, err := ownerRefControllerPatch(m.Controller, m.controllerKind, pod.UID)
    if err != nil {
        return err
    }
    // Patch更新Pod。
    return m.podControl.PatchPod(pod.Namespace, pod.Name, patchBytes)
}

// ReleasePod()发送Patch更新请求到apiserver来释放Pod拥有权，从Pod.OwnerReferences中通过UID删除ControllerRef。
func (m *PodControllerRefManager) ReleasePod(pod *v1.Pod) error {
    klog.V(2).Infof("patching pod %s_%s to remove its controllerRef to %s/%s:%s",
        pod.Namespace, pod.Name, m.controllerKind.GroupVersion(), m.controllerKind.Kind, m.Controller.GetName())
    // deleteOwnerRefStrategicMergePatch()会生成一个删除指定UID的OwnerReference的补丁。
    // 关于Patch更新的实现笔者会有单独的文章介绍，此处只需要知道Patch更新能够删除指定Pod(UID)的指定OwnerReferences(UID)即可。
    patchBytes, err := deleteOwnerRefStrategicMergePatch(pod.UID, m.Controller.GetUID())
    if err != nil {
        return err
    }
    // Patch更新Pod
    err = m.podControl.PatchPod(pod.Namespace, pod.Name, patchBytes)
    if err != nil {
        if errors.IsNotFound(err) {
            // 如果Pod不存在，跟已经释放Pod拥有权是一样的
            return nil
        }
        if errors.IsInvalid(err) {
            // 返回Invalid错误有两种可能：
            // 1. Pod没有OwnerReferences，即为nil；
            // 2. 补丁包中PodUID不存在，即Pod被删除后又创建
            // 这两种可能造成的错误都可以忽略，因为Controller就没有Pod的拥有权，所以返回成功
            return nil
        }
    }
    return err
}
```

## ReplicaSetControllerRefManager

了解PodControllerRefManager，那么ReplicaSetControllerRefManager就非常容易了，它是用来管理ReplicaSet的ControllerRef的。那么问题来了，谁的的子对象会是ReplicaSet？答案是Deployment，所以此处得到一个结论：Deployment对Pod的控制是通过ReplicaSet实现的，那么Deployment控制什么？这个可以参看[DeploymentController](./DeploymentController.md)。

因为有了PodControllerRefManager的铺垫，笔者不会对ReplicaSetControllerRefManager做比较详细的注释，源码链接: <https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/controller/controller_ref_manager.go#L261>。

```go
// ReplicaSetControllerRefManager基本上与PodControllerRefManager相同，只是资源类型的不同。
type ReplicaSetControllerRefManager struct {
    BaseControllerRefManager
    controllerKind schema.GroupVersionKind
    rsControl      RSControlInterface
}

// ReplicaSetControllerRefManager的构造函数，与PodControllerRefManager基本相同，不多解释
func NewReplicaSetControllerRefManager(
    rsControl RSControlInterface,
    controller metav1.Object,
    selector labels.Selector,
    controllerKind schema.GroupVersionKind,
    canAdopt func() error,
) *ReplicaSetControllerRefManager {
    return &ReplicaSetControllerRefManager{
        BaseControllerRefManager: BaseControllerRefManager{
            Controller:   controller,
            Selector:     selector,
            CanAdoptFunc: canAdopt,
        },
        controllerKind: controllerKind,
        rsControl:      rsControl,
    }
}

// ClaimReplicaSets()尝试获取一组ReplicaSet的拥有权，比ClaimPods()少了过滤函数。
func (m *ReplicaSetControllerRefManager) ClaimReplicaSets(sets []*apps.ReplicaSet) ([]*apps.ReplicaSet, error) {
    var claimed []*apps.ReplicaSet
    var errlist []error

    // 匹配函数就是用标签选择器实现的，笔者认为标签选择器实现的匹配函数应该实现在BaseControllerRefManager中，子类直接继承使用。
    match := func(obj metav1.Object) bool {
        return m.Selector.Matches(labels.Set(obj.GetLabels()))
    }
    // 接纳/释放ReplicaSet拥有权的函数。
    adopt := func(obj metav1.Object) error {
        return m.AdoptReplicaSet(obj.(*apps.ReplicaSet))
    }
    release := func(obj metav1.Object) error {
        return m.ReleaseReplicaSet(obj.(*apps.ReplicaSet))
    }

    // 一个接一个的获取ReplicaSet的拥有权
    for _, rs := range sets {
        ok, err := m.ClaimObject(rs, match, adopt, release)
        if err != nil {
            errlist = append(errlist, err)
            continue
        }
        if ok {
            claimed = append(claimed, rs)
        }
    }
    // 返回已经获取拥有权的ReplicaSet和获取拥有权失败的错误
    return claimed, utilerrors.NewAggregate(errlist)
}

// AdoptReplicaSet()和ReleaseReplicaSet()与前面的AdoptPod()和ReleasePod()基本一样，笔者不在重复注释了
func (m *ReplicaSetControllerRefManager) AdoptReplicaSet(rs *apps.ReplicaSet) error {
    if err := m.CanAdopt(); err != nil {
        return fmt.Errorf("can't adopt ReplicaSet %v/%v (%v): %v", rs.Namespace, rs.Name, rs.UID, err)
    }
    patchBytes, err := ownerRefControllerPatch(m.Controller, m.controllerKind, rs.UID)
    if err != nil {
        return err
    }
    return m.rsControl.PatchReplicaSet(rs.Namespace, rs.Name, patchBytes)
}

func (m *ReplicaSetControllerRefManager) ReleaseReplicaSet(replicaSet *apps.ReplicaSet) error {
    klog.V(2).Infof("patching ReplicaSet %s_%s to remove its controllerRef to %s/%s:%s",
        replicaSet.Namespace, replicaSet.Name, m.controllerKind.GroupVersion(), m.controllerKind.Kind, m.Controller.GetName())
    patchBytes, err := deleteOwnerRefStrategicMergePatch(replicaSet.UID, m.Controller.GetUID())
    if err != nil {
        return err
    }
    err = m.rsControl.PatchReplicaSet(replicaSet.Namespace, replicaSet.Name, patchBytes)
    if err != nil {
        if errors.IsNotFound(err) {
            return nil
        }
        if errors.IsInvalid(err) {
            return nil
        }
    }
    return err
}
```

## ControllerRevisionControllerRefManager

ControllerRevisionControllerRefManager笔者留给读者自己阅读，因为ReplicaSetControllerRefManager已经有大量的重复内容了。
