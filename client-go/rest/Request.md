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

## NewRequest

在[RESTClient解析](./Client.md)中实现Interface章节，笔者提到了核心就是Request的构造函数，现在就来看看Request构造函数的实现，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/request.go#L118>

```go
// NewRequest()是Request的构造函数，需要传入RESTClient对象。
func NewRequest(c *RESTClient) *Request {
    // 创建Request的退避管理器，关于退避管理器的功能已经在RESTClient中解释很多了，此处不再赘述了。
    // 只需要知道一点，每个Request都有一个独立的退避管理器。
    var backoff BackoffManager
    if c.createBackoffMgr != nil {
        backoff = c.createBackoffMgr()
    }
    // 如果RESTClient没有设置创建退避管理器的函数，则不使用退避策略。
    if backoff == nil {
        backoff = noBackoff
    }

    // 设置Request的URL路径前缀，就是RESTClient.base.Path/RESTClient.versionedAPIPath
    // 需要注意的是，一般RESTClient.base.Path是空的，而https://192.168.1.2:6443分散在RESTClient.base.Scheme和RESTClient.base.Host中。
    // 所以此处的pathPrefix与c.versionedAPIPath基本是相同的，例如/apis/apps/v1
    var pathPrefix string
    if c.base != nil {
        pathPrefix = path.Join("/", c.base.Path, c.versionedAPIPath)
    } else {
        pathPrefix = path.Join("/", c.versionedAPIPath)
    }

    // 复用http.Client的超时作为请求的超时
    var timeout time.Duration
    if c.Client != nil {
        timeout = c.Client.Timeout
    }

    // 创建Request对象
    r := &Request{
        c:              c,
        rateLimiter:    c.rateLimiter,
        backoff:        backoff,
        timeout:        timeout,
        pathPrefix:     pathPrefix,
        // 默认最多尝试次数是10次
        maxRetries:     10,
        warningHandler: c.warningHandler,
    }

    // 设置Accept请求头，优先使用AcceptContentTypes，如果没有设置则使用ContentType
    // 此处如果对AcceptContentTypes和ContentType不了解，请参看笔者解析RESTClient的文章。
    switch {
    case len(c.content.AcceptContentTypes) > 0:
        r.SetHeader("Accept", c.content.AcceptContentTypes)
    case len(c.content.ContentType) > 0:
        r.SetHeader("Accept", c.content.ContentType+", */*")
    }
    return r
}
```

现在应该理解为什么通过RESTClient创建Request了，因为RESTClient包含了大量Request之间共享的信息，比如限流器、退避管理器构造函数、路径前缀，更关键的是Request.c指向了RESTClient，这样Request就具备了向服务端提交请求的能力。

## Setter/Getter

在Request的构造函数中有很多成员变量没有初始化，比如verb、namespace、resource等，这些都是与具体的请求相关的了。Request提供了大量的Setter/Getter接口用来设置和获取这些成员变量，因为大部分接口过于简单，笔者不一一介绍，只挑选一些重点的接口函数进行解析。

首先来看看Request是如何设置body的，例如向apiserver提交创建Deployment的请求，就需要将Deployment对象放入到Request的body中。而Request.body是io.Reader类型，这里面就有编码的过程，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/request.go#L425>

```go
// Body()用于设置Request.body成员变量，有没有发现该接口的返回值是自己，这种设计可以优雅的实现连续Setter接口的调用，例如：
// request := RESTClient.Get().Namespace(ns).Resource("pods").SubResource("proxy").Name(net.JoinSchemeNamePort(scheme, name, port)).Suffix(path)
// 而不是这样
// request := RESTClient.Get()
// request.Namespace(ns)
// request.Resource("pods")
// ...
// 但是这种实现也有一个缺点，那就是无法返回错误，出错调用者感知不到，只能将错误记录在Request中，这就是Request.err存在的原因。
// 还要Body()传入的参数是interface{}，不是runtime.Object，这让Request的使用范围非常广。
func (r *Request) Body(obj interface{}) *Request {
    // 在这之前调用某个接口(多半是Setter)的时候已经出错了，所以就没必要再继续执行，直接返回即可。
    if r.err != nil {
        return r
    }
    // 现在我们来看看interface{}到底比runtime.Object好在哪里！
    switch t := obj.(type) {
    // 字符串类型，必须是文件的路径，这一点需要注意，也就是说可以将某个文件的内容写入Request.body()。
    // 这种用法是不是很熟悉？是不是感觉kubectl create -f ...可以用这种方法实现？
    case string:
        data, err := ioutil.ReadFile(t)
        if err != nil {
            r.err = err
            return r
        }
        glogBody("Request Body", data)
        r.body = bytes.NewReader(data)
    // []byte类型，直接转为io.Reader还是比较容易的，适用于已经将对象序列化成[]byte的情况
    case []byte:
        glogBody("Request Body", t)
        r.body = bytes.NewReader(t)
    // io.Reader本尊，直接使用即可，适用于已经将对象转换为io.Reader的情况
    case io.Reader:
        r.body = t
    // 终于到Kubernetes的API对象了。
    case runtime.Object:
        // 需要校验obj是否为空，这个有点意思了，为什么不用if nil == obj呢？这就考验读者的语言基本功了。
        // 1. 如果obj为nil是不会进入这个分支的，说明obj != nil
        // 2. interface{}其实就是一个指针二元组(typeptr, objectptr)，分别指向对象的类型和对象本身，所以obj不为空是正常的
        // 3. switch type语法无非是把interface{}类型转换为runtime.Object类型，t依然还是一个(typeptr, objectptr)的二元组，所以t也不是nil
        // 所以此处只能通过反射的方法获取对象指针来判断指向的对象是不是空地址，那么问题来了，为什么我们平时编程的时候不这么判断？
        // 那是因为我们平时编程很难造出这种类型的对象，大部分代码中都明确了对象的类型和指针，除非你这样写：
        // obj := (*appsv1.Deployment)(unsafe.Pointer(nil))，此时的obj就是一个*appsv1.Deployment类型但是实际指向了一个nil对象。
        // 有没有看到unsafe关键字，就是告诉使用者这是不安全的，如果没有足够的把握建议不要使用，但是在Kubernetes里面还是有人这么用的。
        if reflect.ValueOf(t).IsNil() {
            return r
        }
        // 根据配置的内容类型获得编码器，如果不理解可以把encoder想象为json.Marshal()函数。
        encoder, err := r.c.content.Negotiator.Encoder(r.c.content.ContentType, nil)
        if err != nil {
            r.err = err
            return r
        }
        // 编码对象，其实就是序列化对象，此处可以直接看做json.Marshal(obj)，这样好理解一点
        data, err := runtime.Encode(encoder, t)
        if err != nil {
            r.err = err
            return r
        }
        // 将编码后的对象设置为body
        glogBody("Request Body", data)
        r.body = bytes.NewReader(data)
        // 设置Content-Type请求头，告知apiserver编码类型
        r.SetHeader("Content-Type", r.c.content.ContentType)
    // 其他类型的obj不知道怎么处理，所以报错
    default: 
        r.err = fmt.Errorf("unknown type used for body: %+v", obj)
    }
    return r
}
```

有没有发现Request没有一个成员变量记录最终的URL？这是因为很多成员变量没有设置是无法知道最终的URL的。而构造http.Request需要传入最终的URL，这就是需要Request提供获取最终URL的接口，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/request.go#L468>

```go
// URL()函数将生成Request的最终URL，例如https://192.168.1.2:6443/apis/apps/v1/namespaces/ns/deployments/name/subresource/subpath?params
func (r *Request) URL() *url.URL {
    // URL路径以前缀作为起始，例如/apis/apps/v1
    p := r.pathPrefix
    // 如果Request设置了命名空间，则在路径上追加命名空间，需要注意的是增加了一个"namespaces"，例如/apis/apps/v1/namespaces/ns
    if r.namespaceSet && len(r.namespace) > 0 {
        p = path.Join(p, "namespaces", r.namespace)
    }
    // 如果Request设置了资源种类(deployments)，则追加资种类，此处需要注意的是将资源种类转换为小写。
    // 例如/apis/apps/v1/namespaces/ns/deployments/，此处需要注意资源(Resource)和种类(Kind)是不同的，
    // 所以是deployments而不是deployment，这部分会在Kubernetes API约定中有介绍。
    if len(r.resource) != 0 {
        p = path.Join(p, strings.ToLower(r.resource))
    }
    // 如果Request设置了资源名或者子路径或者子资源，则追加到路径中，例如/apis/apps/v1/namespaces/ns/deployments/name/subresources/subpath.
    // 此处的资源名就是Deployment.Name
    if len(r.resourceName) != 0 || len(r.subpath) != 0 || len(r.subresource) != 0 {
        p = path.Join(p, r.resourceName, r.subresource, r.subpath)
    }

    // 准备返回最终URL
    finalURL := &url.URL{}
    if r.c.base != nil {
        *finalURL = *r.c.base
    }
    finalURL.Path = p

    // 将Request的请求参数转换为url.Values类型，因为url.Values类型具有URL编码能力
    query := url.Values{}
    for key, values := range r.params {
        for _, value := range values {
            query.Add(key, value)
        }
    }

    // 特殊处理一下超时，超时是作为请求参数的一部分
    if r.timeout != 0 {
        query.Set("timeout", r.timeout.String())
    }
    // 编码参数并赋值给最终URL
    finalURL.RawQuery = query.Encode()
    return finalURL
}
```
