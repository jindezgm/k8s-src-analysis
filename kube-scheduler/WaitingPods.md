<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-20 11:42:10
 * @Description: 等待Pod源码解析
-->

# 前言

本文的等待Pod不是[调度队列](./SchedulingQueue.md)中等待调度的Pod，在调度队列中的Pod类型是PodInfo。本文的等待Pod是在[调度插件](./Plugin.md)中PermitPlugin返回等待的Pod，虽然当前没有PermitPlugin的实现，但这不影响我们理解kube-scheduler如何处理多个插件同时返回等待的情况。毕竟有任何插件返回拒绝，Pod就会返回调度队列，全部返回批准，Pod就会进入绑定周期，所以kube-scheduler需要管理至少一个PermitPlugin返回等待并且其他PermitPlugin都返回批准的Pod

本文引用kubernetes源码release-1.21分支。

# WaitingPod

WaitingPod是一个接口类型，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/interface.go#L253>

```go
type WaitingPod interface {
    // 获取等待Pod的API对象，很好理解，一个等待Pod应该继承v1.Pod
    GetPod() *v1.Pod
    // 获取哪些PermitPlugin插件需要Pod等待。
    GetPendingPlugins() []string
    // 名字为pluginName的插件批准了Pod，如果这是最后一个批准的插件，那么该Pod就可以进入绑定周期了
    Allow(pluginName string)
    // 拒绝等待的Pod，Pod需要返回调度队列
    Reject(msg string)
}
```

## WaitingPod实现

WaitingPod的实现是waitingPod,源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/runtime/waiting_pods_map.go#L73>

```go
type waitingPod struct {
    // Pod的API对象，WaitingPod.GetPod()直接返回'pod'就可以了。
    pod            *v1.Pod
    // key是要求Pod等待的插件的名字，WaitingPod.GetPendingPlugins()就是把map的key转为slice。
    // value是定时器，即等待截止时间，至于超时后做什么见下文的构造函数
    pendingPlugins map[string]*time.Timer
    // 用于阻塞等待WaitingPod结果的协程，同时可以将结果返回给等待协程。所有的插件都批准或者任何待超时都会向s输出结果。
    s              chan *framework.Status
    mu             sync.RWMutex
}
```

来看看WaitingPod的构造函数，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/runtime/waiting_pods_map.go#L83>

```go
// newWaitingPod()是WaitingPod的构造函数。
func newWaitingPod(pod *v1.Pod, pluginsMaxWaitTime map[string]time.Duration) *waitingPod {
    wp := &waitingPod{
        pod: pod,
        // 缓冲设置为1非常好理解，它的效果等同于sync.Cond，生产者不会被阻塞，同时还能保证输出结果不丢失
        s: make(chan *framework.Status, 1),
    }

    wp.pendingPlugins = make(map[string]*time.Timer, len(pluginsMaxWaitTime))
    
    wp.mu.Lock()
    defer wp.mu.Unlock()
    for k, v := range pluginsMaxWaitTime {
        // 根据插件返回的等待时间设置定时器，等待超时就拒绝等待的Pod
        plugin, waitTime := k, v
        wp.pendingPlugins[plugin] = time.AfterFunc(waitTime, func() {
            msg := fmt.Sprintf("rejected due to timeout after waiting %v at plugin %v",
                waitTime, plugin)
            wp.Reject(plugin, msg)
        })
    }

    return wp
}
```

从waitingPod定义的成员变量很容易知道GetPod()和GetPendingPlugins()的实现，没有任何知识点。本文只解析一下Allow()和Reject()的实现，源码链接：<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/runtime/waiting_pods_map.go#L130>。

```go
// Allow()实现了WaitingPod.Allow()接口。
func (w *waitingPod) Allow(pluginName string) {
    w.mu.Lock()
    defer w.mu.Unlock()
    // 如果指定插件存在，则停止计时器并删除它，因为该插件已经批准Pod了。
    if timer, exist := w.pendingPlugins[pluginName]; exist {
        timer.Stop()
        delete(w.pendingPlugins, pluginName)
    }

    // 如果还有插件需要等待则直接返回，即等待Pod的协程依然被阻塞
    if len(w.pendingPlugins) != 0 {
        return
    }

    // 输出等待结果，Success表示所有插件都批准了，即便此时没有协程接收s的结果(即获取该Pod等待结果)也没有问题，因为s的缓冲为1所以不会丢失结果。
    // 既然缓冲都是1了，为什么还有default这个case？因为Allow可能会与某个插件超时同时发生。
    // 也就是说Allow()和Reject()可能存在并发调用的可能，这会造成后调用者被阻塞。
    select {
    case w.s <- framework.NewStatus(framework.Success, ""):
    default:
    }
}

// Reject()实现了WaitingPod.Reject()接口。
func (w *waitingPod) Reject(pluginName, msg string) {
    w.mu.RLock()
    defer w.mu.RUnlock()
    // 停止所有的插件定时器，因为这个Pod已经被拒绝了
    for _, timer := range w.pendingPlugins {
        timer.Stop()
    }

    // 输出等待结果，Unschedulable表示Pod不可调度，需要返回调度队列。
    select {
    case w.s <- framework.NewStatus(framework.Unschedulable, msg).WithFailedPlugin(pluginName):
    default:
    }
}
```

# waitingPodsMap

kube-scheduler中WaitingPod可能会很多，立刻能想到的方法就是用一个map管理起来，再加一个锁保护一下。嗯！这就是waitingPodsMap，<https://github.com/kubernetes/kubernetes/blob/release-1.21/pkg/scheduler/framework/runtime/waiting_pods_map.go#L30>。

```go
// 注意：waitingPodsMap用Pod的UID做键，而不是NS+Name
type waitingPodsMap struct {
    pods map[types.UID]*waitingPod
    mu   sync.RWMutex
}

// 下面的代码是在没有任何注释的必要，放在这里省的读者点连接进去看了，内容不多，也占不了多少版面。
func (m *waitingPodsMap) add(wp *waitingPod) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.pods[wp.GetPod().UID] = wp
}

func (m *waitingPodsMap) remove(uid types.UID) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.pods, uid)
}
func (m *waitingPodsMap) get(uid types.UID) *waitingPod {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.pods[uid]
}

func (m *waitingPodsMap) iterate(callback func(framework.WaitingPod)) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    for _, v := range m.pods {
        callback(v)
    }
}
```

# 总结

1. 只有WaitingPod的实现有点内容，waitingPodsMap基本可以忽略不计；
2. waitingPod用map\[string\]*time.Timer实现了多个PermitPlugin的等待，所有的PermitPlugin批准才能通过，任何time.Timer超时都会拒绝；
3. waitingPod通过一个结果(状态值)chan来阻塞获取Pod等待结果的协程并返回等待结果，其中chan的缓冲大小为1，既保证了生产者不会阻塞，同时保证了结果不丢失；
