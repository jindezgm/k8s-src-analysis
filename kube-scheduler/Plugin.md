<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-09 21:47:58
 * @Description: 
-->

# 前言

下图是维基百科关于[Plug-in (computing)](https://en.wikipedia.org/wiki/Plug-in_%28computing%29)示例插件框架，关于插件机制实现的原理不了解的读者可以看看，本文不赘述了。
![插件示例图](./Plug-In.png)

但是上图中几个知识点本文会提及，所以此处重点说明一下：

1. 宿主应用(Host Application): kube-scheduler就是宿主应用，因为所有插件都是给kube-scheduler使用的。
2. 插件管理器(Plugin Manager): 原理上讲Scheduling Profile(后文会有介绍)充当了插件管理器的角色，因为它可以配置使能/禁止插件的使用。
3. 插件接口(Plugin Interface): kube-scheduler抽象了调度框架（framework），将调度分为不同的阶段，每个阶段都定了一种插件接口。
4. 服务接口(Services Interface): kube-scheduler在创建插件的时候传入了FrameworkHandle，就是一种服务接口，插件通过FrameworkHandle可以获取Clientset和SharedInformerFactory，或者从Cache中读取数据。该句柄还将提供列举、批准或拒绝等待的Pod的接口。

本文内容参考了[Scheduling Framework](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md)，并结合源码解析kube-scheduler的插件实现。

# 插件状态

kube-scheduler中插件是无状态的，如果插件需要存储状态，则依赖外部实现。这就需要宿主应用为插件提供相关的能力，并以服务接口的形式开放给插件，比如插件A想引用插件B的输出数据。kube-scheduler的实现方法略有不同，开放给插件一个变量，该变量会在所有插件接口函数中引用，其实开放服务接口和开放变量本质上是一样的。现在来看看存储插件状态的类型定义：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/cycle_state.go#L44>

```go
// CycleState就是开放给所有插件共享使用的变量类型，从定义来看就是一个加了锁的map，可以理解为所有插件共享一个KV存储。
// 其中Cycle是什么意思？很简单，就是每调度一个Pod算一轮，那么CycleState就是某一轮的状态。
type CycleState struct {
    mx      sync.RWMutex
    // StateKey和StateData的定义下面有注释
    // map[StateKey]StateData其实等同于map[string]interface{}，而这个interface{}具有克隆能力。
    // 这给插件实现非常大的扩展空间，基本想存什么就存什么。
    storage map[StateKey]StateData
    // 但凡有Metrics关键字，八九不离十都和prometheus的监控相关
    recordPluginMetrics bool
}

// StateKey就是字符串
type StateKey string

// StateData就是具有拷贝能力（可以是深度复制，也可以不是）的interface{}
type StateData interface {
    Clone() StateData
}
```
