<!--
 * @Author: jinde.zgm
 * @Date: 2021-04-10 12:04:18
 * @Description: client-go的RestRequest源码解析
-->

# 前言

在阅读本文之前建议先阅读[Kubernetes RestClient解析](./Client.md)。如果把RestClient类比为http.Client，那么本文的主角Request则可以类比为http.Request。与RestClient直接继承http.Client功能不同，Request并没有继承http.Request的功能，而是随时可以根据Request构造http.Request对象。

还有一个与http.Request核心区别需要注意，那就是http.Request只是一个请求，它需要通过http.Client才能提交给服务端，即http.Client.Do(http.Request)。但是Kubernetes的Request修改了这个编程逻辑，赋予了Request提交请求给服务端的能力，即Request.Do()。不要小看这一点点的修改，这改变的是编程思想，意味着所有的Request都是活的，不需要显式通过RestClient就可以发送给服务端。这简化了使用者的开发，因为只需要一个Request对象就可以了。Request不仅可以用来提交请求，还实现了Watch()接口，即监视指定路径下的API对象，接下来笔者就来解析Request是如何实现这些吊炸天的功能。

`本文引用源码为github.com/kubernetes/client-go的release-1.21分支。`

# Request

先来看看Request的定义，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/request.go#L88>

```go
type Request struct {
    // Request不需要显式依赖RESTClient的方法就是将RESTClient作为成员变量。
    c *RESTClient

    // warning等级异常处理器，指向了RESTClient.warningHandler，所有的Request共享使用
    warningHandler WarningHandler

    // 限速器，指向了RESTClient.rateLimiter，Request之间共享使用
    rateLimiter flowcontrol.RateLimiter
    // 退避管理器，Request独享使用，也就是说该Request退避去其他请求无关
    backoff     BackoffManager
    // 请求超时时间
    timeout     time.Duration
    // 请求最多尝试次数
    maxRetries  int

    // HTTP请求的动词，比如POST、GET、DELETE、PATCH、PUT等
    verb       string
    // 资源路径前缀，其实就是RESTClient.base/RESTClient.versionedAPIPath，详情参看Request构造函数
    pathPrefix string
    // 子路径，这是与子资源相关的，详情参看笔者关于API的解析文章。
    subpath    string
    // HTTP请求的参数，就是请求URL后面的那些参数，例如https://xxxxx?a=b&c=d
    params     url.Values
    // HTTP请求头，最终会赋值给http.Request.Header
    headers    http.Header

    // 以下这些变量是Kubernetes API约定的一部分
    // 关于Kubernetes API约定参看：https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md
    // 请求需要操作API对象的命名空间
    namespace    string
    // 命名空间是否已经设定的标记，直接判断len(namespaces) == 0不就完了？此处需要知道的是，空命名空间是Kubernetes保留使用的。
    // 所以有没有设置命名空间和命名空间是否为空不是一回事。
    namespaceSet bool
    // 请求操作API对象的种类，比如Pod
    resource     string
    // 请求操作API对象的名字，比如Pod.Name
    resourceName string
    // 请求操作API对象的子资源，比如bind、proxy
    subresource  string

    // 请求的输出变量，包括错误代码和HTTP响应body
    err  error
    body io.Reader
}
```