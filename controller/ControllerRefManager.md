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