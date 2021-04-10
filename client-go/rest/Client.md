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

// ClientContentConfig是客户端关于HTTP内容(http.Request.Body和http.Response.Body)的配置。
type ClientContentConfig struct {
    // 指定客户端将接受的内容类型，如果没有设置，将使用ContentType设置Accept请求头。就是RestClient希望apiserver响应内容的类型。
    // 关于Accept请求头参看：https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept
    AcceptContentTypes string
    // 指定用于与服务端通信的格式，如果未设置AcceptContentTypes，则将用于设置Accept请求头，并设置为给服务端的任何对象的默认内容类型。
    // 如果没有设置，则使用"application/json"。就是告诉服务端(apiserver)http.Request.Body的格式。
    ContentType string
    // API组和版本，直接初始化RestClient时必须提供。
    GroupVersion schema.GroupVersion
    // Negotiator用于获取支持的多种媒体类型的编码器和解码器，关于Negotiator读者可以暂时不用深入理解，只需要知道他可以根据媒体类型获取编解码器就可以了。
    // 至于什么是编解码器，说白了就是用来序列化和反序列化API对象的，可以直接理解为json.Marshal()和json.Unmarsal()。
    // 因为AcceptContentTypes和ContentType两个配置，所以需要根据相应的类型获取编解码器。
    Negotiator runtime.ClientNegotiator
}
```

看为了RestClient的定义，接下来看看RestClient的构造函数，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/client.go#L107>

```go
// NewRESTClient()是RestClient的构造函数，从构造函数的参数上看，RestClient大部分成员变量都是外部传入的。
func NewRESTClient(baseURL *url.URL, versionedAPIPath string, config ClientContentConfig, rateLimiter flowcontrol.RateLimiter, client *http.Client) (*RESTClient, error) {
    // ContentType如果没有设置，默认使用"application/json"，默认的编码器和解码器也可以看做是json.Marshal()和json.Unmarshal()。
    if len(config.ContentType) == 0 {
        config.ContentType = "application/json"
    }

    // 初始化baseURL，这个比较好理解。
    base := *baseURL
    if !strings.HasSuffix(base.Path, "/") {
        base.Path += "/"
    }
    base.RawQuery = ""
    base.Fragment = ""

    // 返回RestClient对象
    return &RESTClient{
        base:             &base,
        versionedAPIPath: versionedAPIPath,
        content:          config,
        // readExpBackoffConfig是构造退避管理器的函数，下面有注释
        createBackoffMgr: readExpBackoffConfig,
        rateLimiter:      rateLimiter,

        Client: client,
    }, nil
}

// readExpBackoffConfig()是RestClient构造退避管理器的函数
func readExpBackoffConfig() BackoffManager {
    // 通过环境变量获取退避的初始时间和最大时间，单位为秒。
    backoffBase := os.Getenv(envBackoffBase)
    backoffDuration := os.Getenv(envBackoffDuration)

    // 因为环境变量是字符串型，所以此处需要转为整型
    backoffBaseInt, errBase := strconv.ParseInt(backoffBase, 10, 64)
    backoffDurationInt, errDuration := strconv.ParseInt(backoffDuration, 10, 64)
    // 如果没有配置环境变量或者错误配置，则无需退避
    if errBase != nil || errDuration != nil {
        return &NoBackoff{}
    }
    // URLBackoff是以URL为键计算退避时间，也就是说某个URL访问错误，如果立刻访问相同的URL就需要退避一段时间，退避时间随着错误次数增加以2的指数增加。
    // 此处需要注意的是，URL是baseURL，这个应该好理解，某个主机访问出错，立刻访问这个主机大概率还是可能会出错，即便访问的是不同的API。
    return &URLBackoff{
        Backoff: flowcontrol.NewBackOff(
            time.Duration(backoffBaseInt)*time.Second,
            time.Duration(backoffDurationInt)*time.Second)}
}
```

看完了RestClient的定义，接下来看看RestClient的构造函数，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/client.go#L169>

```go
// Verb()创建了Request对象，然后设置Request的verb属性。
func (c *RESTClient) Verb(verb string) *Request {
    return NewRequest(c).Verb(verb)
}

// Post()利用Verb()实现，这个非常简单。
func (c *RESTClient) Post() *Request {
    return c.Verb("POST")
}

// Put()利用Verb()实现，这个非常简单。
func (c *RESTClient) Put() *Request {
    return c.Verb("PUT")
}

// Patch()利用Verb()创建了Request，然后将Patch类型设置Content-Type请求头，读者可以看下types.PatchType的枚举定义就更容易理解了。
func (c *RESTClient) Patch(pt types.PatchType) *Request {
    return c.Verb("PATCH").SetHeader("Content-Type", string(pt))
}

// Get()利用Verb()实现，这个非常简单。
func (c *RESTClient) Get() *Request {
    return c.Verb("GET")
}

// Delete()利用Verb()实现，这个非常简单。
func (c *RESTClient) Delete() *Request {
    return c.Verb("DELETE")
}

// 获取API组和版本。
func (c *RESTClient) APIVersion() schema.GroupVersion {
    return c.content.GroupVersion
}
```

感觉RestClient实现Interface接口非常简单，其实这里面最核心的函数是NewRequest()，利用RestClient构造Request对象，笔者会在[解析Request的文章](./Request.md)详细说明。

# 总结

1. RestClient是操作某个组/版本(比如apps/v1)的API对象的REST客户端；
2. RestClient的主要功能就是创建Request，甚至可以理解为RequestFactory；
3. RestClient的限流器是所有Request共享使用的，也就是说RestClient创建的Request虽然可以并发提交，但是都会被限流器统一限制在某个QPS以内；
4. RestClient的构造函数传入了限流器，不是自己构造的，可以推测多个RestClient(比如apps/v1、core/v1)之间共享同一个限流器；
5. 退避管理器是以URL(base)为键管理退避时间，同一个Request向不同的host提交产生的错误不会累加，单独计算退避时间；
