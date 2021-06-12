<!--
 * @Author: jinde.zgm
 * @Date: 2021-06-12 09:14:04
 * @Description: Rest Config解析
-->


# 前言

[RESTClient](./Client.md#RESTClient)的文章中关于构造函数部分，虽然清晰的解析了构造函数的实现，但是对于构造函数的参数来源并没有给出答案。同时在总结部分推测多个RESTClient(比如apps/v1、core/v1)之间共享同一个限流器，本文将针对这部分内容给出答案。这就需要深度解析配置（Config）的实现，Kubernetes利用配置创建RESTClient，这很好理解，也是我们日常变成中比较常见的方法。

`本文引用源码为github.com/kubernetes/client-go的release-1.21分支。`

# Config

我们平时一般用一个结构体描述配置，Kubernetes也不例外，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/config.go#L53>

```go
// 在了解Config之前，笔者需要在这里简单回顾一下RESTClient构造函数的参数，因为Config就是用来配置RESTClient构造函数的参数：
// 1. baseURL(url.URL): 基础URL，比如https://192.168.1.2:6443
// 2. versionedAPIPath(string): API组/版本的路径，比如/apis/app/v1
// 3. config(ClientContentConfig): http.Request.Body和http.Response.Body的内容类型以及相关的编解码器
// 4. rateLimiter(flowcontrol.RateLimiter): 限速器
// 5. client(http.Client): HTTP客户端
type Config struct {
    // Host必须是一个主机字符串，host:port(比如https://192.168.1.2:6443)或一个指向apiserver基础的URL。
    // 简单一句话就是apiserver的URL，Host是API对象的全URL的一部分，用来定位apiserver的位置。
    // Host最终会通过类型转换成(string->url.URL)为baseURL
	Host string
    // API根路径，不同的API组可能在不同的根目录下，大部分API组都在/apis路径下，core组的API在/api路径下。
    // 如果说Host用来定位apiserver的位置，那么APIPath用来定位API组在apiserver的哪个路径下。
    // 此处为什么强调API组？因为RESTClient是某个API组的客户端，所以APIPath是某个API组在apiserver的根路径。
	APIPath string

    // 关于HTTP内容(http.Request.Body和http.Response.Body)的配置。
    // 与RESTClient文章中介绍ClientContentConfig相似，最终需要转换为ClientContentConfig来创建RESTClient.
	ContentConfig

    // 用户名和密码，大部分情况应该不使用这种认证方式。
    // Config中有很多成员变量是用来创建hppt.Client的，比如Username、Password。
    // 本文的目标是分析如何利用Config创建RESTClient，虽然http.Client也是创建RESTClient的必须参数之一。
    // 但是通过Config创建http.Client的部分笔者会单独在transport相关文章中进行解析，本文可以假设使用http.DefaultClient即可。
	Username string
	Password string `datapolicy:"password"`

    // 创建http.Client相关的参数，暂不注释。
	BearerToken string `datapolicy:"token"`

    // 创建http.Client相关的参数，暂不注释。
	BearerTokenFile string

	// 创建http.Client相关的参数，暂不注释。
	Impersonate ImpersonationConfig

	// 创建http.Client相关的参数，暂不注释。
	AuthProvider *clientcmdapi.AuthProviderConfig

	// 创建http.Client相关的参数，暂不注释。
	AuthConfigPersister AuthProviderConfigPersister

	// 创建http.Client相关的参数，暂不注释。
	ExecProvider *clientcmdapi.ExecConfig

    // TLS配置，创建http.Client相关的参数，暂不注释。
	TLSClientConfig

    // 创建http.Client相关的参数，暂不注释。
    // UserAgent请求头参看：https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/User-Agent
	UserAgent string

	// 创建http.Client相关的参数，暂不注释。
	DisableCompression bool

	// 创建http.Client相关的参数，暂不注释。
	Transport http.RoundTripper
	// 创建http.Client相关的参数，暂不注释。
	WrapTransport transport.WrapperFunc

    // QPS指示从此客户端到主服务端的最大QPS。如果为零，则创建的RESTClient将使用默认QPS(5)
	QPS float32

    // 与QPS一起用于创建限流器，RESTClient限流器采用令牌桶算法，Burst是一次可以获取的最大令牌数，感兴趣的读者可以自己看下相关实现。
    // 如果为零，则创建的RESTClient将使用默认Burst(10)。
	Burst int

    // 限速器，如果设置了限速器则覆盖QPS/Burst构造的限速器
	RateLimiter flowcontrol.RateLimiter

    // warning等级异常处理器，赋值给在RESTClient，所有的Request共享使用。
    // Kubernetes默认的WarningHandler实现就是简单的写日志，后文有注释。
	WarningHandler WarningHandler

    // 等待超时时间，零值表示没有超时。
	Timeout time.Duration

    // 创建http.Client相关的参数，暂不注释。
	Dial func(ctx context.Context, network, address string) (net.Conn, error)

	// 创建http.Client相关的参数，暂不注释。
	Proxy func(*http.Request) (*url.URL, error)
}
```

从Config的定义可以看出大部分都是用来创建http.Client的，这也间接的说明创建http.Client是一个相对复杂的过程，所以笔者单独一系列文章解析Kubernetes是如何创建http.Client的，相信能够学到不少东西。

## RESTClientFor

前面已经知道了Config，接下来我们来看看是如何利用Config创建RESTClient的，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/config.go#L306>

```go
// RESTClientFor()根据配置创建RESTClient
func RESTClientFor(config *Config) (*RESTClient, error) {
    // 必须配置API组/版本，因为RESTClient用来操作指定组/版本的API对象，不理解的话可以阅读笔者的RESTClient文章
	if config.GroupVersion == nil {
		return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
	}
    // 必须配置NegotiatedSerializer(不好翻译)，因为提交http.Request和处理http.Response需要API对象的编码器和解码器
	if config.NegotiatedSerializer == nil {
		return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
	}

    // defaultServerUrlFor()函数本文就不详细注释了，可以简单理解为以下两步：
    // 1. baseURL将config.Host字符串转换为url.URL类型，可以理解为一次类型装换，这个是创建RESTClient需要的;
    // 2. versionedAPIPath=/config.APIPath/config.GroupVersion.Group/config.GroupVersion.Version，比如/apis/apps/v1
	baseURL, versionedAPIPath, err := defaultServerUrlFor(config)
	if err != nil {
		return nil, err
	}

    // 创建http.Client，这是一个比较复杂的过程，此处不做详细说明。
	transport, err := TransportFor(config)
	if err != nil {
		return nil, err
	}

	var httpClient *http.Client
	if transport != http.DefaultTransport {
		httpClient = &http.Client{Transport: transport}
		if config.Timeout > 0 {
			httpClient.Timeout = config.Timeout
		}
	}

    // 如果配置了限速器就用配置的限速器，否则根据QPS和Burst创建一个限速器
	rateLimiter := config.RateLimiter
	if rateLimiter == nil {
		qps := config.QPS
		if config.QPS == 0.0 {
            // 如果没有配置QPS则使用默认值5
			qps = DefaultQPS
		}
		burst := config.Burst
		if config.Burst == 0 {
            // 如果没有配置Burst则使用默认值100
			burst = DefaultBurst
		}
		if qps > 0 {
            // 创建一个令牌桶算法的限速器，感兴趣的读者可以了解一下。
            // 需要注意的是，新建的限速器并没有赋值给config，所以一点没有配置限速器，则RESTClient自己独享一个限速器。
			rateLimiter = flowcontrol.NewTokenBucketRateLimiter(qps, burst)
		}
	}

    // 创建ClientContentConfig，如果看过了RESTClient的文章这部分就很好理解了。
	var gv schema.GroupVersion
	if config.GroupVersion != nil {
		gv = *config.GroupVersion
	}
	clientContent := ClientContentConfig{
		AcceptContentTypes: config.AcceptContentTypes,
		ContentType:        config.ContentType,
		GroupVersion:       gv,
		Negotiator:         runtime.NewClientNegotiator(config.NegotiatedSerializer, gv),
	}

    // 创建RESTClient
	restClient, err := NewRESTClient(baseURL, versionedAPIPath, clientContent, rateLimiter, httpClient)
	if err == nil && config.WarningHandler != nil {
        // 为RESTClient设置WarningHandler
		restClient.warningHandler = config.WarningHandler
	}
	return restClient, err
}
```

上面的代码虽然将Config.WarningHandler赋值给了RESTClient，但是Config.WarningHandler可能是nil。如果RESTClient.warningHandler为nil，将使用默认的WarningHandler，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/rest/warnings.go#L64>

```go
// WarningLogger实现了WarningHandler，是默认的WarningHandler，当然Kubernetes也提供了设置默认WarningHandler的接口。
type WarningLogger struct{}

// HandleWarningHeader()实现了WarningHandler.HandleWarningHeader()接口。
func (WarningLogger) HandleWarningHeader(code int, agent string, message string) {
    // 只处理299的状态码，说明这些报警都使用299状态码，然后用message区分
	if code != 299 || len(message) == 0 {
		return
	}
    // 就是将message打印到日志中
	klog.Warning(message)
}
```

笔者在RESTClient和Request的文章中只是简单的提了一下WarningHandler，并没有解释到底有什么功能，本文给出了默认的实现，就是写日志。当前还有其他的WarningHandler实现，笔者就不一一注释了，毕竟不是本文的重点内容。

通过Config创建RESTClient除了创建http.Client部分，其他都还比较好理解，但是笔者还有一个问题，就是到底有没有配置限速器呢？这就需要追溯到Clientset，Clientset是所有API组/版本RESTClient的集合，在Clientset的构造函数中需要创建每个API组/版本的RESTClient。后续章节将通过Clientset的构造函数解析Kubernetes是如何通过Config创建RESTClient。

## NewForConfig

kubernetes.NewForConfig()是Client的构造函数，此处加了包名是避免与后文同名的函数混淆，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/kubernetes/clientset.go#L394>

```go
func NewForConfig(c *rest.Config) (*Clientset, error) {
    // 报备Config，是因为函数会修改Config，这样可以避免修改用户传入的参数。
    // configShallowCopy变量名字有点意思，特意强调ShallowCopy而非DeepCopy，这二者的不同读者真的了解么？
	configShallowCopy := *c
    // 如果没有配置限速器，同时设置了QPS，则创建限速器，结合RESTClientFor()函数，总结如下：
    // 1. Config.RateLimiter == nil && Config.QPS > 0，则Clientset创建一个Config.QPS的限速器，所有RESTClient共享；
    // 2. Config.RateLimiter == nil && Config.QPS == 0，则每个API组/版本的RESTClient自己创建一个默认QPS为5的限速器；
	if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS > 0 {
		if configShallowCopy.Burst <= 0 {
			return nil, fmt.Errorf("burst is required to be greater than 0 when RateLimiter is not set and QPS is set to greater than 0")
		}
		configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
	}
	var cs Clientset
	var err error
    // 下面是Clientset创建每个API组/版本的RESTClient，比较简单，就不在注释了
	cs.admissionregistrationV1, err = admissionregistrationv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.admissionregistrationV1beta1, err = admissionregistrationv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.internalV1alpha1, err = internalv1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.appsV1, err = appsv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.appsV1beta1, err = appsv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.appsV1beta2, err = appsv1beta2.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.authenticationV1, err = authenticationv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.authenticationV1beta1, err = authenticationv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.authorizationV1, err = authorizationv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.authorizationV1beta1, err = authorizationv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.autoscalingV1, err = autoscalingv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.autoscalingV2beta1, err = autoscalingv2beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.autoscalingV2beta2, err = autoscalingv2beta2.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.batchV1, err = batchv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.batchV1beta1, err = batchv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.certificatesV1, err = certificatesv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.certificatesV1beta1, err = certificatesv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.coordinationV1beta1, err = coordinationv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.coordinationV1, err = coordinationv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.coreV1, err = corev1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.discoveryV1, err = discoveryv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.discoveryV1beta1, err = discoveryv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.eventsV1, err = eventsv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.eventsV1beta1, err = eventsv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.extensionsV1beta1, err = extensionsv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.flowcontrolV1alpha1, err = flowcontrolv1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.flowcontrolV1beta1, err = flowcontrolv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.networkingV1, err = networkingv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.networkingV1beta1, err = networkingv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.nodeV1, err = nodev1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.nodeV1alpha1, err = nodev1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.nodeV1beta1, err = nodev1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.policyV1, err = policyv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.policyV1beta1, err = policyv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.rbacV1, err = rbacv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.rbacV1beta1, err = rbacv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.rbacV1alpha1, err = rbacv1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.schedulingV1alpha1, err = schedulingv1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.schedulingV1beta1, err = schedulingv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.schedulingV1, err = schedulingv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.storageV1beta1, err = storagev1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.storageV1, err = storagev1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.storageV1alpha1, err = storagev1alpha1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}

	cs.DiscoveryClient, err = discovery.NewDiscoveryClientForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	return &cs, nil
}
```

Clientset中虽然很多RESTClient对象，但是创建方法都是千篇一律，本文只挑选创建core/v1的RESTClient的代码作为代表，其他的读者可以自己阅读，源码链接：<https://github.com/kubernetes/client-go/blob/release-1.21/kubernetes/typed/core/v1/core_client.go#L117>

```go
// NewForConfig()创建core/v1的RESTClient，需要注意的是CoreV1Client继承了rest.Interface，而RESTClient是rest.Interface的唯一实现。
// 所以创建CoreV1Client基本等同于创建RESTClient。
func NewForConfig(c *rest.Config) (*CoreV1Client, error) {
	// 设置默认配置，这个有意思了，传入的配置还需要设置什么默认配置？难道说有什么配置参数还没有设置不成？详情见下面注释。
	config := *c
	if err := setConfigDefaults(&config); err != nil {
		return nil, err
	}
	// 根据Config创建RESTClient
	client, err := rest.RESTClientFor(&config)
	if err != nil {
		return nil, err
	}
	// 返回CoreV1Client对象
	return &CoreV1Client{client}, nil
}

// setConfigDefaults()设置默认配置
func setConfigDefaults(config *rest.Config) error {
	// 哈哈，这下明白默认配置是啥了，每个API的组/版本不同，而Clientset使用同一个Config创建所有的RESTClient，
	// 所以需要每个API组/版本设置自己的相关配置，没毛病。
	gv := v1.SchemeGroupVersion
    // 设置API的组/版本
	config.GroupVersion = &gv
    // core/v1的API路径是/api，apps/v1的API路径是/apis
	config.APIPath = "/api"
	config.NegotiatedSerializer = scheme.Codecs.WithoutConversion()

	// 如果没有设置用户代理，Kubernetes有默认的用户代理，就是用应用程序名、版本、操作系统、体系架构以及git提交ID构建的。
	if config.UserAgent == "" {
		config.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	return nil
}
```

# 总结

1. Config如果配置了限速器，则Clientset中的所有RESTClient共享该限速器
2. Config如果没有配置限速器并且QPS大于0，则Clientset中的所有RESTClient共享一个限速器，该限速器的QPS等于Config.QPS
3. Config如果没有配置限速器并且QPS为0，则Clientset中的所有RESTClient都有一个独享的限速器，QPS为默认值5；

至于Config到底有没有配置限速器和QPS?这个就要看具体的应用(比如kubelet、kube-scheduler)了。
