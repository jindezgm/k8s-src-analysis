<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-04 21:27:36
 * @Description: 
-->

# 前言

ListerWatcher是Lister和Watcher的结合体，前者负责列举全量对象，后者负责监视（本文将watch翻译为监视）对象的增量变化。为什么要有这个接口？原因很简单，提交访问效率。众所周知，kubernetes所有API对象都存储在etcd中，并只能通过apiserver访问。如果很多客户端频繁的列举全量对象（比如列举所有的Pod），这会造成apiserver不堪重负。那么如果在客户端做本地缓存如何？至少在没有任何状态变化的情况下只需要读取本地缓存即可，效率提升显而易见。通过列举全量对象完成本地缓存，而监视增量则是为了及时的将apiserver的状态变化更新到本地缓存。所以，在apiserver与客户端之间绝大部分传输的是对象的增量变化，当然在异常的情况下还是要重新列举一次全量对象。

本文值得客户端本地缓存就是[Indexer](./Indexer.md)，client-go不仅实现了缓存，同时还加了索引，进一步提升了检索效率。

本文采用kubernetes的release-1.20分支。

# ListerWatcher定义

## Lister定义

源码连接：<https://github.com/kubernetes/client-go/blob/release-1.20/tools/cache/listwatch.go#L30>

```go
type Lister interface {
    // metav1.ListOptions和runtime.Object的定义在apimachinery目录，此处不做相关说明
    // 只需要知道列举要有选项，返回的是一个列表对象，runtime.Object既可以是单个API对象，也可以是API列表对象
    List(options metav1.ListOptions) (runtime.Object, error)
}
```

## Watcher定义

源码连接：<https://github.com/kubernetes/client-go/blob/release-1.20/tools/cache/listwatch.go#L37>

```go
type Watcher interface {
    // 需要指定监视的起始版本，一般是列举全量对象的最大版本号+1
    Watch(options metav1.ListOptions) (watch.Interface, error)
}
```

## ListerWatcher定义

源码连接：<https://github.com/kubernetes/client-go/blob/release-1.20/tools/cache/listwatch.go#L43>

```go
// 没什么好说的
type ListerWatcher interface {
    Lister
    Watcher
}
```

# ListerWatcher实现

源码连接：<https://github.com/kubernetes/client-go/blob/release-1.20/tools/cache/listwatch.go#L57>

```go
// ListFunc是对ListerWatcher.List()接口的函数类型的定义
type ListFunc func(options metav1.ListOptions) (runtime.Object, error)

// WatchFunc是对ListerWatcher.Watch()接口的函数类型的定义
type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)

// ListWatch实现了ListerWatcher，它方便了使用者实现ListerWatcher，因为只需要设置两个函数就可以了。
// 比如传入闭包，后面ListerWatcher应用的案例就是使用的闭包，ListFunc和WatchFunc不能为空。
type ListWatch struct {
    ListFunc  ListFunc
    WatchFunc WatchFunc
    // 全量列举对象的时候禁止分页，但是没看到有设置为true的地方，等同于没用
    DisableChunking bool
}
// List实现了ListerWatcher.List()
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
    return lw.ListFunc(options)
}

// Watch实现了ListerWatcher.Watch()
func (lw *ListWatch) Watch(options metav1.ListOptions) (watch.Interface, error) {
    return lw.WatchFunc(options)
}
```

# ListerWatcher应用

ListerWatcher主要用于创建各种API对象的SharedIndexInformer，本文只截取Pod对象的代码，其他API对象都是一样的，只是API对象类型的差异。源码连接：<https://github.com/kubernetes/client-go/blob/release-1.20/informers/core/v1/pod.go#L58>

```go
// NewFilteredPodInformer用来创建Pod类型的SharedIndexInformer，其中kubernetes.Interface用来实现ListerWatcher
// kubernetes.Interface是啥？但是它的实现kubernetes.Clientset应该很熟悉了吧，所以ListerWatcher就是使用kubernetes.Clientset各个资源的List和Watch函数实现的
// 这样Clientset和SharedIndexInformer之间的就联系起来了，至于函数的其他参数的说明请阅读相关的文档。
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
    return cache.NewSharedIndexInformer(
        &cache.ListWatch{
            // ListFunc就是Clientset的List函数
            ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
                if tweakListOptions != nil {
                    tweakListOptions(&options)
                }
                return client.CoreV1().Pods(namespace).List(context.TODO(), options)
            },
            // WatchFunc就是Clientset的Watch函数
            WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
                if tweakListOptions != nil {
                    tweakListOptions(&options)
                }
                return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
            },
        },
        &corev1.Pod{},
        resyncPeriod,
        indexers,
    )
}
```

# 总结

1. ListerWatcher就是为SharedIndexInformer结局全量对象、监视对象增量变化设计的接口，实现就是Clientset的List和Watch函数；
2. SharedIndexInformer利用ListerWatcher实现了本地缓存与apiserver之间的状态一致性；
3. 不仅可以提升客户端访问API对象的效率，同时可以将对象的增量变化回调给使用者；
4. 从原理上讲，可以用etcd的clientv3.Client实现ListerWatcher，SharedIndexInformer同步etcd的对象，这样一些简单的醒目就可以复用SharedIndexInformer了，毕竟不是所有的项目都需要一个apiserver；