<!--
 * @Author: jinde.zgm
 * @Date: 2021-03-31 22:09:43
 * @Description: JobController源码解析
-->

# 前言

关于Job的介绍参看[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)，笔者提取几个关键的信息：

1. Job(实际上是JobController)会创建一个或者多个Pod，如何指定创建多少个Pod?
2. 直到指定数量的Pod成功终止，如何制定多少个Pod成功终止？
3. 在有些情况下，希望Job在经历若干次失败重试之后直接进入失败状态，如何指定重试次数？

以上问题在[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)都有答案，本文将从Job的API定义以及JobController源码实现的角度看看Kubernetes如何实现Job的功能的。

本文引用源码为kubernetes的release-1.21版本
