<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-09 21:47:58
 * @Description: 
-->

# 前言

下图是危机百科关于[Plug-in (computing)](https://link.zhihu.com/?target=http%3A//en.wikipedia.org/wiki/Plug-in_%28computing%29)实例插件框架，关于插件机制实现的原理不了解的读者可以看看，本文不赘述了。
![插件示例图](./Plug-in.png)

但是上图中几个知识点本文会提及，所以此处重点说明一下：

1. 宿主应用(Host Application): kube-scheduler就是宿主应用，因为所有插件都是给kube-scheduler使用的；
2. 插件管理器(Plugin Manager): 
3. 插件接口(Plugin Interface): kube-scheduler将调度抽象为一种框架，调度框架为每个阶段都定义了一种插件接口；
4. 服务接口(Services Interface): 

# 插件状态

kube-scheduler中插件是无状态的，如果插件需要存储状态，则依赖外部实现。这就需要宿主应用为插件提供相关的能力，并以服务接口的形式开放给插件，例如插件A想引用插件B的输出数据。kube-scheduler的实现方法略有不同，开放给插件一个变量，该变量会在所有插件接口函数中引用，其实开放服务接口和开放变量本质上是一样的。

现在来看看存储插件状态的类型定义：<https://github.com/kubernetes/kubernetes/blob/release-1.20/pkg/scheduler/framework/cycle_state.go#L44>

```go
type CycleState struct {
    mx      sync.RWMutex
    storage map[StateKey]StateData
    // if recordPluginMetrics is true, PluginExecutionDuration will be recorded for this cycle.
    recordPluginMetrics bool
}
```
