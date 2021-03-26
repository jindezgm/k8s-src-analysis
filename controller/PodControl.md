<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-26 22:05:02
 * @Description: PodControl源码解析
-->

# 前言

阅读本文前请参看[名词解释](./README.md)。

PodControl是XxxController操作Pod的接口，比如ReplicaSetController创建、删除Pod。熟悉Clientset的读者肯定会质疑有必要再定义接口来操作Pod么？直接用Clientset提供的接口不就可以了么？答案肯定是有必要的，否则笔者就没必要介绍它了。因为很多Controller的子对象都是Pod，比如Job、Deamonset、ReplicaSet等等。这些Controller对于Pod都有类似的操作，比如设置Pod的拥有者(这一点不同于用kubectl create方式创建的Pod)。所以就有了PodControl接口提供给多个XxxController使用。

关于拥有者的介绍可以参看[ControllerRefManager](./ControllerRefManager.md)。

本文引用源码为kubernetes的release-1.20分支。

# PodControl

## PodControlInterface

PodControlInterface定义了操作Pod的接口，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L449>

```go
type PodControlInterface interface {
    // 根据Pod模板创建Pod，这是XxxController创建Pod的共性，因为很多Workload都有Pod模板。
    CreatePods(namespace string, template *v1.PodTemplateSpec, object runtime.Object) error
    // CreatePodsOnNode()与CreatePods()一样根据Pod模板创建Pod，同时还指定了Pod运行的Node以及拥有者。
    // 遗憾的是CreatePodsOnNode()与CreatePods()都没有被XxxController所使用。
    CreatePodsOnNode(nodeName, namespace string, template *v1.PodTemplateSpec, object runtime.Object, controllerRef *metav1.OwnerReference) error
    // CreatePodsWithControllerRef()与CreatePods()一样根据Pod模板创建Pod，同时指定了拥有者。
    // 这是XxxController大量使用的接口，也证明了笔者前面的观点，即XxxController创建Pod是有共性的。
    CreatePodsWithControllerRef(namespace string, template *v1.PodTemplateSpec, object runtime.Object, controllerRef *metav1.OwnerReference) error
    // 删除一个Pod，这个接口与Clientset的接口没什么不同，无非多了个object参数。
    // 这也是XxxController操作Pod的另一个共性，就是为Pod的父对象记录操作Pod的事件。
    DeletePod(namespace string, podID string, object runtime.Object) error
    // Patch更新Pod。
    PatchPod(namespace, name string, data []byte) error
}
```

## RealPodControl

RealPodControl实现了(仅此一个实现)PodControlInterface接口，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L464>。

```go
type RealPodControl struct {
    // 利用Clientset操作Pod
    KubeClient clientset.Interface
    // 用于记录Pod父对象操作Pod的事件
    Recorder   record.EventRecorder
}
```

### CreatePodXxx

虽然PodControlInterface定义了3个创建Pod的接口，但是笔者搜索发现只有CreatePodsWithControllerRef()被多大量使用，另外两个没有被使用。不排除早期版本中有使用，这并没有什么问题，因为这三个接口都是通过一个函数实现的。源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L521>。

```go
// CreatePods()实现了PodControlInterface.CreatePods()接口。
func (r RealPodControl) CreatePods(namespace string, template *v1.PodTemplateSpec, object runtime.Object) error {
    // createPods()下面有注释。
    return r.createPods("", namespace, template, object, nil)
}

// CreatePodsWithControllerRef()实现了PodControlInterface.CreatePodsWithControllerRef()接口。
func (r RealPodControl) CreatePodsWithControllerRef(namespace string, template *v1.PodTemplateSpec, controllerObject runtime.Object, controllerRef *metav1.OwnerReference) error {
    // 校验controllerRef的合法性
    if err := validateControllerRef(controllerRef); err != nil {
        return err
    }
    return r.createPods("", namespace, template, controllerObject, controllerRef)
}

// CreatePodsOnNode()实现了PodControlInterface.CreatePodsOnNode()接口。
func (r RealPodControl) CreatePodsOnNode(nodeName, namespace string, template *v1.PodTemplateSpec, object runtime.Object, controllerRef *metav1.OwnerReference) error {
    if err := validateControllerRef(controllerRef); err != nil {
        return err
    }
    return r.createPods(nodeName, namespace, template, object, controllerRef)
}

// createPods()是所有创建Pod接口的最终实现。
func (r RealPodControl) createPods(nodeName, namespace string, template *v1.PodTemplateSpec, object runtime.Object, controllerRef *metav1.OwnerReference) error {
    // 根据模板创建Pod的API对象，GetPodFromTemplate()下面有注释。
    pod, err := GetPodFromTemplate(template, object, controllerRef)
    if err != nil {
        return err
    }
    // 是否指定了Node，对应于CreatePodsOnNode()接口。
    if len(nodeName) != 0 {
        pod.Spec.NodeName = nodeName
    }
    // 不能创建没有标签的Pod，为什么？因为Controller需要根据Pod标签匹配，如果是Controller创建的Pod必须都有标签。
    if len(labels.Set(pod.Labels)) == 0 {
        return fmt.Errorf("unable to create pods, no labels")
    }
    // 通过Clientset创建Pod，有没有发现PodControl创建Pod的接口都没有metav1.CreateOptions？
    // 这也算是PodControl存在的另一个价值，省去了没用的参数。
    newPod, err := r.KubeClient.CoreV1().Pods(namespace).Create(context.TODO(), pod, metav1.CreateOptions{})
    if err != nil {
        if !apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
            // 记录为object创建Pod失败事件
            r.Recorder.Eventf(object, v1.EventTypeWarning, FailedCreatePodReason, "Error creating: %v", err)
        }
        return err
    }
    // 获取object对象的meta来打印日志
    accessor, err := meta.Accessor(object)
    if err != nil {
        klog.Errorf("parentObject does not have ObjectMeta, %v", err)
        return nil
    }
    // 记录为object创建Pod成功事件。
    klog.V(4).Infof("Controller %v created pod %v", accessor.GetName(), newPod.Name)
    r.Recorder.Eventf(object, v1.EventTypeNormal, SuccessfulCreatePodReason, "Created pod: %v", newPod.Name)

    return nil
}

// 根据Pod模板创建Pod的API对象。
func GetPodFromTemplate(template *v1.PodTemplateSpec, parentObject runtime.Object, controllerRef *metav1.OwnerReference) (*v1.Pod, error) {
    // 从模板中提取Labels、Finalizers和Annotations
    desiredLabels := getPodsLabelSet(template)
    desiredFinalizers := getPodsFinalizers(template)
    desiredAnnotations := getPodsAnnotationSet(template)
    accessor, err := meta.Accessor(parentObject)
    // 获取Pod父对象的meta
    if err != nil {
        return nil, fmt.Errorf("parentObject does not have ObjectMeta, %v", err)
    }
    // 利用父对象的名字+'-'作为Pod的前缀，这也是为什么我们看到Deployment创建出来的Pod都是xxx-yyy-zzz格式。
    // 其中xxx是Deployment名字，yyy是ReplicaSet名字，zzz是Pod名字，也就是Deployment创建了ReplicatSet，ReplicatSet创建了Pod
    prefix := getPodsPrefix(accessor.GetName())

    // 常见Pod的API对象
    pod := &v1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Labels:       desiredLabels,
            Annotations:  desiredAnnotations,
            GenerateName: prefix,
            Finalizers:   desiredFinalizers,
        },
    }
    // 如果指定了Pod的Controller引用，将其追加到Pod的拥有者引用中
    if controllerRef != nil {
        pod.OwnerReferences = append(pod.OwnerReferences, *controllerRef)
    }
    // 从Pod模板深度拷贝PodSpec。
    pod.Spec = *template.Spec.DeepCopy()
    return pod, nil
}
```

### DeletePod

这个接口也没什么好解释的，直接上代码吧，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L599>。

```go
// DeletePod()实现了PodControlInterface.DeletePod()接口。
func (r RealPodControl) DeletePod(namespace string, podID string, object runtime.Object) error {
    获取Pod父对象的meta，单纯为了打印日志，这应该是也是XxxController操作Pod的共性吧。
    accessor, err := meta.Accessor(object)
    if err != nil {
        return fmt.Errorf("object does not have ObjectMeta, %v", err)
    }
    klog.V(2).InfoS("Deleting pod", "controller", accessor.GetName(), "pod", klog.KRef(namespace, podID))
    // 通过Clientset删除Pod
    if err := r.KubeClient.CoreV1().Pods(namespace).Delete(context.TODO(), podID, metav1.DeleteOptions{}); err != nil {
        if apierrors.IsNotFound(err) {
            klog.V(4).Infof("pod %v/%v has already been deleted.", namespace, podID)
            return err
        }
        // 为object记录删除Pod失败的事件
        r.Recorder.Eventf(object, v1.EventTypeWarning, FailedDeletePodReason, "Error deleting: %v", err)
        return fmt.Errorf("unable to delete pods: %v", err)
    }
    // 为object记录删除Pod成功的事件
    r.Recorder.Eventf(object, v1.EventTypeNormal, SuccessfulDeletePodReason, "Deleted pod: %v", podID)

    return nil
}
```

### PatchPod

Patch更新接口比Clientset的接口要简洁一些，少了一些看不懂的参数，这也是PodControl存在的价值，<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/controller/controller_utils.go#L539>。

```go
// PatchPod()实现了PodControlInterface.PatchPod()接口。
func (r RealPodControl) PatchPod(namespace, name string, data []byte) error {
    // PatchPod()接口实现比较简单，直接用Clientset接口实现Patch更新。但是要求所有的XxxController都使用策略性合并(types.StrategicMergePatchType).
    // 为什么不为Patch更新Pod记录事件？这一点笔者也在思考。
    // 关于Patch类型的说明请参看：https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
    _, err := r.KubeClient.CoreV1().Pods(namespace).Patch(context.TODO(), name, types.StrategicMergePatchType, data, metav1.PatchOptions{})
    return err
}
```

# 总结

1. PodControl是Clientset操作Pod接口的再封装，但是在接口上去掉了一些对于XxxController'无用'的参数，比如metav1.PatchOptions、metav1.DeleteOptions、metav1.CreateOptions等，同时增加了XxxController需要的参数，比如父对象、ControllerRef等；
2. PodControl能够为Pod的父对象记录操作Pod的事件，避免每个Controller都实现类似的代码；
