<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-31 22:09:43
 * @Description: JobController源码解析
-->

# 前言

关于Job的介绍参看[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)，笔者提取几个关键的信息：

1. Job(实际上是JobController)会创建一个或者多个Pod，如何指定创建多少个Pod?
2. 直到指定数量的Pod成功终止，如何指定多少个Pod成功终止？
3. 在有些情况下，希望Job在经历若干次失败重试之后直接进入失败状态，如何指定重试次数？

以上问题在[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)都有答案，本文将从Job的API定义以及JobController源码实现的角度看看Kubernetes如何实现Job的功能的。

本文引用源码为kubernetes的release-1.21版本

# Job(API)

Job的API定义了Job的参数(配置、规格都行，反正意思到了就行)和Job的状态，前言中提到了几个问题都可以在API的定义找到答案。源码链接：<https://github.com/kubernetes/api/blob/release-1.21/batch/v1/types.go#L30>。

```go
type Job struct {
    // 继承了TypeMeta和ObjectMeta，这是Kubernete所有的API类型必须要继承的，没什么好解释的
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

    // Job的参数，可以理解为Job的常量，是Job的目标状态，详情参看JobSpec的注释。
    // 当然，Job可能会更新，更新的JobSpec是不会变化的直到下一次Job的更新。
    // 这段时间可能很短也可能很长，笔者将JobSpec定义为常量就是相对于这段时间内的。
    Spec JobSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

    // Job的状态，可以理解为Job的变量，是Job的当前状态，详情参看JobStatus的注释。
    // JobController(后文简称Controller)就是计算JobSpec与JobStatus之间的差异(diff)，根据差异执行一些操作可以将diff变成0.
    // 当然执行的某些操作可能会失败，JobSpec在执行这些操作的过程中也可能发生变化，Controller就是无限的计算差异并修复差异。
    Status JobStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```
