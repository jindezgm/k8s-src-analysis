<!--
 * @Author: jinde.zgm
 * @Date: 2021-04-09 21:06:32
 * @Description: client-go的RestClient源码解析
-->

# 前言

apiserver的REST客户端可以类比为http.Client，至少它继承了http.Client的功能(更准确的说应该是有一个http.Client的成员变量)，如果读者对http.Client不了解，那么建议先简单学习一下，这非常有助于理解本文的内容。

为了区分apiserver的REST客户端和http.Client，本文将用RestClient表示apiserver的REST客户端，而http.Client则直接引用齐全名。因为apiserver是一个相对标准的REST服务，所以访问apiserver的客户端最终都要通过http.Client实现，那么RestClient继承http.Client的功能就很好理解了(笔者注：此处用继承功能而不是继承，是出于概念严谨考虑的，因为继承功能的方法很多，比如采用成员变量亦或是直接集成)。http.Client只是具备http客户端基本功能，Kubernetes对客户端还提出了其他需求，所以就有了RestClient，具体是什么需求，后续章节会给出答案。

`本文引用源码为github.com/kubernetes/client-go的release-1.21分支。`

# Client

## Interface

Interface是Kubernetes对REST客户端抽象的接口，从Interface的定义我们可以看出Kubernetes对REST于客户端的需求，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/client.go#L41>

```go
type Interface interface {
    // 获取限速器，关于限速器读者可以暂时不用深入理解，只需要知道它是用来控制客户端向服务端提交请求的QPS即可。
    // 通过这个接口笔者可以得出一个结论：Kubernetes要求RestClient有限速能力，这个是http.Client不具备的。
    // 限速的目的很简单，避免某个客户端恶意/异常操作对apiserver造成压力，进而影响整个系统。
    GetRateLimiter() flowcontrol.RateLimiter
    // 创建指定动词的REST请求(Request)，这个有点意思了，说明Kubernetes的REST请求是可以RestClient创建的。
    // 熟悉http.Request的同学都知道，http.Request是不需要依赖http.Client创建的。
    // 通过这个接口笔者可以得出一个结论：Kubernetes要求RestClient有创建Request的能力，这也是http.Client不具备的。
    Verb(verb string) *Request
    // 等同于Verb("POST")
    Post() *Request
    // 等同于Verb("PUT")
    Put() *Request
    // 等同于Verb("PATCH")
    Patch(pt types.PatchType) *Request
    // 等同于Verb("GET")
    Get() *Request
    // 等同于Verb("DELETE")
    Delete() *Request
    // 获取API组和版本，Kubernetes的每个RestClient只能操作某个组/版本的API对象。
    // 这也就可以理解为什么需要RestClient创建Request，因为需要创建的Request操作范围指定在某个API组/版本内。
    // 是不是可以想象得到Clientset的实现？其实就是所有API组/版本的RestClient集合，所以称之为Clientset。
    APIVersion() schema.GroupVersion
}
```

通过对Interface接口的分析，一句话概括为：“RestClient用于创建操作指定组/版本API对象的Request，同时具有限速能力”。总结的是不是很到位？那么问题来了，只需要创建Request，这跟http.Client一毛钱关系都没有，那为啥前文说可以类比http.Client呢？你想啊，创建Request不是目的，目的是将Request提交给apiserver，那靠啥提交请求呢？Interface只是接口，不会反应具体的实现，它只是需求提出方。要想知道原因，且看下文的解析。

## RestClient

RestClient确实有这个类型，而且实现了Interface接口，笔者前言就开始使用RestClient也是为了思路的连贯性，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/client.go#L81>

```go
// RestClient实现了Interface接口，所有的成员变量除了限速器和APIVersion，其他的基本都是为了创建Request而设计的。
type RESTClient struct {
    // 客户端所有调用的根URL，比如https://192.168.1.2:6443，是生成Request的最终URL的必须参数
    base *url.URL
    // 资源路径的一段，连接base就是资源的根路径，比如/apis/apps/v1(对应于apps组v1版本，/apis是绝大部分API的前缀)
    // versionedAPIPath是生成Reqeust的最终URL的必须参数
    versionedAPIPath string

    // 与HTTP内容相关的配置，下面有注释
    content ClientContentConfig

    // 退避管理器的构造函数，首先需要知道什么是退避，说的简单点就是向apiserver提交请求后出错，如果需要重试则需要等待一段时间，这就是退避。
    // 其次需要知道退避管理器的功能是什么？退避管理器有点类似于map[string]int，用来记录不同对象(键)的退避次数，然后根据退避次数计算退避时间。
    // 至于退避管理器的键是什么，后面会有介绍，此处只需要知道向apiserver提交请求可能会失败，失败可能需要重试，重试可能需要计算退避时间。
    // createBackoffMgr就是用来给Request创建自己的退避管理器的。
    createBackoffMgr func() BackoffManager

    // 限速器，首先是为了实现Interface.GetRateLimiter()接口，其次，所有创建的Request共享限速器。
    // 所以rateLimiter用来限制RestClient创建的Request向apiserver提交请求的QPS。至于flowcontrol.RateLimiter的实现感兴趣的读者可以自己了解一下。
    rateLimiter flowcontrol.RateLimiter

    // warningHandler，而不是errorHandler，用来处理一些warning等级的异常，这些异常不会终止程序执行。
    // warningHandler也是Request之间共享使用的。
    warningHandler WarningHandler

    // 这个应该没什么好解释的了，提交HTTP请求必须的对象，如果没有设置将使用http.DefaultClient。
    // 前文提到RestClient继承了http.Client就是通过成员变量实现的。
    Client *http.Client
}
```
