<!--
 * @Author: jinde.zgm
 * @Date: 2021-05-31 22:29:54
 * @Description: Serializer解析
-->

# 前言

序列化和反序列化在很多项目中都有应用，Kubernetes也不例外。Kubernetes中定义了大量的API对象，为此还单独设计了一个[包](https://github.com/kubernetes/api)，方便多个模块引用。API对象在不同的模块之间传输(尤其是跨进程)可能会用到序列化与反序列化，不同的场景对于序列化个格式又不同，比如grpc协议用protobuf，用户交互用yaml(因为yaml可读性强)，etcd存储用json。Kubernetes反序列化API对象不同于我们常用的json.Unmarshal()函数(需要传入对象指针)，Kubernetes需要解析对象的类型(Group/Version/Kind)，根据API对象的类型构造API对象，然后再反序列化。因此，Kubernetes定义了Serializer接口，专门用于API对象的序列化和反序列化。

本文引用源码为kubernetes的release-1.21分支。

# Serializer

因为Kubernetes需要支持json、yaml、protobuf三种数据格式的序列化和反序列化，有必要抽象序列化和反序列化的统一接口，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/interfaces.go#L86>

```go
// Serializer是用于序列化和反序列化API对象的核心接口，
type Serializer interface {
    // Serializer继承了编码器和解码器，编码器就是用来序列化API对象的，序列化的过程称之为编码；反之，反序列化的过程称之为解码。
    // 关于编/解码器的定义下面有注释。
	Encoder
	Decoder
}

// 序列化的过程称之为编码，实现编码的对象称之为编码器(Encoder)
type Encoder interface {
    // Encode()将对象写入流。可以将Encode()看做为json(yaml).Marshal()，只是输出变为io.Writer。
	Encode(obj Object, w io.Writer) error

    // Identifier()返回编码器的标识符，当且仅当两个不同的编码器编码同一个对象的输出是相同的，那么这两个编码器的标识符也应该是相同的。
    // 也就是说，编码器都有一个标识符，两个编码器的标识符可能是相同的，判断标准是编码任意API对象时输出都是相同的。
    // 标识符有什么用？标识符目标是与CacheableObject.CacheEncode()方法一起使用，CacheableObject又是什么东东？后面有介绍。
	Identifier() Identifier
}

// 标识符就是字符串，可以简单的理解为标签的字符串形式，后面会看到如何生成标识符。
type Identifier string

// 反序列化的过程称之为解码，实现解码的对象称之为解码器(Decoder)
type Decoder interface {
    // Decode()尝试使用Schema中注册的类型或者提供的默认的GVK反序列化API对象。
    // 如果'into'非空将被用作目标类型，接口实现可能会选择使用它而不是重新构造一个对象。
    // 但是不能保证输出到'into'指向的对象，因为返回的对象不保证匹配'into'。
    // 如果提供了默认GVK，将应用默认GVK反序列化，如果未提供默认GVK或仅提供部分，则使用'into'的类型补全。
	Decode(data []byte, defaults *schema.GroupVersionKind, into Object) (Object, *schema.GroupVersionKind, error)
}
```

我们平时工作中最常用的序列化格式包括json、yaml以及protobuf，非常"巧合"，Kubernetes也是用这几种序列化格式。以json为例，编码器和解码器可以等同于json.Marshal()和json.Unmarshal()，定义成interface是对序列化与反序列化的统一抽象。为什么我们平时很少抽象（当然有些读者是有抽象的，我们不能一概而论），是因为我们工作中可能只用到一种序列化格式，所以抽象显得没那么必要。而Kubernetes中，这三种都是需要的，yaml的可视化效果好，比如我们写的各种yaml文件；而API对象存储在etcd中是json格式，在用到grpc的地方则需要protobuf格式。

再者，Kubernetes对于Serializer的定义有更高的要求，即根据序列化的数据中的元数据自动识别API对象的类型(GVK)，这在Decoder.Decode()接口定义中已经有所了解。而我们平时使用json.Marshal()的时候传入了指定类型的对象指针，相比于Kubernetes对于反序列化的要求，我们使用的相对更"静态"。

综上所述，抽象Serializer就有必要了，尤其是[RecognizingDecoder](./RecognizingDecoder.md)可以解码任意格式的API对象就可以充分体现这种抽象的价值。

## json

json.Serializer实现了将API对象序列化成json数据和从json数据反序列化API对象，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L100>

```go
// Serializer实现了runtime.Serializer接口。
type Serializer struct {
    // MetaFactory从json数据中提取GVK(Group/Version/Kind)，下面有MetaFactory注释。
    // MetaFactory很有用，解码时如果不提供默认的GVK和API对象指针，就要靠MetaFactory提取GVK了。
    // 当然，即便提供了供默认的GVK和API对象指针，提取的GVK的也是非常有用的，详情参看Decode()接口的实现。
	meta    MetaFactory
    // SerializerOptions是Serializer选项，可以看做是配置，下面有注释。
	options SerializerOptions
    // runtime.ObjectCreater根据GVK构造API对象，在反序列化时会用到，其实它就是Schema。
    // runtime.ObjectCreater的定义读者可以自己查看源码，如果对Schema熟悉的读者这都不是事。
	creater runtime.ObjectCreater
    // runtime.ObjectTyper根据API对象返回可能的GVK，也是用在反序列化中，其实它也是Schema。
    // 这个有什么用？runtime.Serializer.Decode()接口注释说的很清楚，在json数据和默认GVK无法提供的类型元数据需要用输出类型补全。
	typer   runtime.ObjectTyper
    // 标识符，Serializer一旦被创建，标识符就不会变了。
	identifier runtime.Identifier
}

// SerializerOptions定义了Serializer的选项.
type SerializerOptions struct {
    // true: 序列化/反序列化yaml；false: 序列化/反序列化json
    // 也就是说，json.Serializer既可以序列化/反序列json，也可以序列化/反序列yaml。
	Yaml bool

    // Pretty选项仅用于Encode接口，输出易于阅读的json数据。当Yaml选项为true时，Pretty选项被忽略，因为yaml本身就易于阅读。
    // 什么是易于阅读的？举个例子就立刻明白了，定义测试类型为：
    // type Test struct {
    //     A int
    //     B string    
    // }
    // 则关闭和开启Pretty选项的对比如下：
    // {"A":1,"B":"2"}
    // {
    //   "A": 1,
    //   "B": "2"
    // }
    // 很明显，后者更易于阅读。易于阅读只有人看的时候才有需要，对于机器来说一点价值都没有，所以这个选项使用范围还是比较有限的。
	Pretty bool

    // Strict应用于Decode接口，表示严谨的。那什么是严谨的？笔者很难用语言表达，但是以下几种情况是不严谨的：
    // 1. 存在重复字段，比如{"value":1,"value":1};
    // 2. 不存在的字段，比如{"unknown": 1}，而目标API对象中不存在Unknown属性;
    // 3. 未打标签字段，比如{"Other":"test"}，虽然目标API对象中有Other字段，但是没有打`json:"Other"`标签
    // Strict选项可以理解为增加了很多校验，请注意，启用此选项的性能下降非常严重，因此不应在性能敏感的场景中使用。
    // 那什么场景需要用到Strict选项？比如Kubernetes各个服务的配置API，对性能要求不高，但需要严格的校验。
	Strict bool
}
```

### MetaFactory

MetaFactory类型名定义其实挺忽悠人的，工厂类都是用来构造对象的，MetaFactory的功能虽然也是构造类型元数据的，但是它更像是一个解析器，所以笔者认为"MetaParser"更加贴切，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/meta.go#L28>

```go
type MetaFactory interface {
    // 解析json数据中的元数据字段，返回GVK(Group/Version/Kind)。
    // 如果MetaFactory就这么一个接口函数，笔者认为叫解释器或者解析器更加合理。
	Interpret(data []byte) (*schema.GroupVersionKind, error)
}

// SimpleMetaFactory是MetaFactory的一种实现，用于检索在json中由"apiVersion"和"kind"字段标识的对象的类型和版本。
type SimpleMetaFactory struct {
}

// Interpret()实现了MetaFactory.Interpret()接口。
func (SimpleMetaFactory) Interpret(data []byte) (*schema.GroupVersionKind, error) {
    // 定义一种只有apiVersion和kind两个字段的匿名类型
	findKind := struct {
		// +optional
		APIVersion string `json:"apiVersion,omitempty"`
		// +optional
		Kind string `json:"kind,omitempty"`
	}{}
    // 只解析json中apiVersion和kind字段，这个玩法有点意思，但是笔者认为这个方法有点简单粗暴。
    // 读者可以尝试阅读json.Unmarshal()，该函数会遍历整个json，开销不小，其实必要性不强，因为只需要apiVersion和kind字段。
    // 试想一下，如果每次反序列化一个API对象都要有一次Interpret()和Decode()，它的开销相当于做了两次反序列化。
	if err := json.Unmarshal(data, &findKind); err != nil {
		return nil, fmt.Errorf("couldn't get version/kind; json parse error: %v", err)
	}
    // 将apiVersion解析为Group和Version
	gv, err := schema.ParseGroupVersion(findKind.APIVersion)
	if err != nil {
		return nil, err
	}
    // 返回API对象的GVK
	return &schema.GroupVersionKind{Group: gv.Group, Version: gv.Version, Kind: findKind.Kind}, nil
}
```

### Decode

其实Serializer的重头戏在解码，因为解码需要考虑的事情比较多，比如提取类型元数据（GVK），根据类型元数据构造API对象，当然还要考虑复用传入的API对象。而编码就没有这么复杂，所以理解了解码的实现，编码就基本可以忽略不计了。源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L209>

```go
// Decode实现了Decoder.Decode()，尝试从数据中提取的API类型(GVK)，应用提供的默认GVK，然后将数据加载到所需类型或提供的'into'匹配的对象中：
// 1. 如果into为*runtime.Unknown，则将提取原始数据，并且不执行解码；
// 2. 如果into的类型没有在Schema注册，则使用json.Unmarshal()直接反序列化到'into'指向的对象中；
// 3. 如果'into'不为空且原始数据GVK不全，则'into'的类型(GVK)将用于补全GVK;
// 4. 如果'into'为空或数据中的GVK与'into'的GVK不同，它将使用ObjectCreater.New(gvk)生成一个新对象;
// 成功或大部分错误都会返回GVK，GVK的计算优先级为originalData > default gvk > into.
func (s *Serializer) Decode(originalData []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	data := originalData
    // 如果配置选项为yaml，则将yaml格式转为json格式，是不是有一种感觉：“卧了个槽”！所谓可支持json和yaml，就是先将yaml转为json！
    // 那我感觉我可以支持所有格式，都转为json就完了呗！其实不然，yaml的使用场景都是需要和人交互的地方，所以对于效率要求不高(qps低)。
    // 那么实现简单易于维护更重要，所以这种实现并没有什么毛病。
	if s.options.Yaml {
        // yaml转json
		altered, err := yaml.YAMLToJSON(data)

		if err != nil {
			return nil, nil, err
		}
		data = altered
	}
    // 此时data是json了，以后就不用再考虑yaml选项了

    // 解析类型元数据，原理已经在MetaFactory解释过了。
	actual, err := s.meta.Interpret(data)
	if err != nil {
		return nil, nil, err
	}

    // 解析类型元数据大部分情况是正确的，除非不是json或者apiVersion格式不对。
    // 但是GVK三元组可能有所缺失，比如只有Kind，Group/Version，其他字段就用默认的GVK补全。
    // 这也体现出了原始数据中的GVK的优先级最高，其次是默认的GVK。gvkWithDefaults()函数下面有注释。
	if gvk != nil {
		*actual = gvkWithDefaults(*actual, *gvk)
	}

    // 如果'into'是*runtime.Unknown类型，需要返回*runtime.Unknown类型的对象。
    // 需要注意的是，此处复用了'into'指向的对象，因为返回的类型与'into'指向的类型完全匹配。
	if unk, ok := into.(*runtime.Unknown); ok && unk != nil {
        // 输出原始数据
		unk.Raw = originalData
        // 输出数据格式为JSON，那么问题来了，不是'data'才能保证是json么？如s.options.Yaml == true，originalData是yaml格式才对。
        // 所以笔者有必要提一个issue，看看官方怎么解决，如此明显的一个问题为什么没有暴露出来？
        // 笔者猜测：runtime.Unknown只在内部使用，而内部只用json格式，所以自然不会暴露出来。
		unk.ContentType = runtime.ContentTypeJSON
        // 输出GVK。
		unk.GetObjectKind().SetGroupVersionKind(*actual)
		return unk, actual, nil
	}

    // 'into'不为空，通过into类型提取GVK，这样在原始数据中的GVK和默认GVK都没有的字段用into的GVK补全。
	if into != nil {
        // 判断'into'是否为runtime.Unstructured类型
		_, isUnstructured := into.(runtime.Unstructured)
        // 获取'into'的GVK，需要注意的是返回的是一组GVK，types[0]是推荐的GVK。
		types, _, err := s.typer.ObjectKinds(into)
		switch {
        // 'into'的类型如果没有被注册或者为runtime.Unstructured类型，则直接反序列成'into'指向的对象。
        // 没有被注册的类型自然无法构造对象，而非结构体等同于map[string]interface{}，不可能是API对象(因为API对象必须是结构体)。
        // 所以这两种情况直接反序列化到'into'对象就可以了，此时与json.Unmarshal()没什么区别。
		case runtime.IsNotRegisteredError(err), isUnstructured:
			if err := caseSensitiveJSONIterator.Unmarshal(data, into); err != nil {
				return nil, actual, err
			}
			return into, actual, nil
        // 获取'into'类型出错，多半是因为不是指针
		case err != nil:
			return nil, actual, err
        // 用'into'的GVK补全未设置的GVK，所以GVK的优先级:originalData > default gvk > into
		default:
			*actual = gvkWithDefaults(*actual, types[0])
		}
	}

    // 如果没有Kind
	if len(actual.Kind) == 0 {
		return nil, actual, runtime.NewMissingKindErr(string(originalData))
	}
    // 如果没有Version
	if len(actual.Version) == 0 {
		return nil, actual, runtime.NewMissingVersionErr(string(originalData))
	}
    // 那么问题来了，为什么不判断Group？Group为""表示"core"，比如我们写ymal的时候Kind为Pod，apiVersion是v1，并没有设置Group。

    // 从函数名字可以看出复用'into'或者重新构造对象，复用的原则是：如果'into'注册的一组GVK有任何一个与*actual相同，则复用'into'。
    // runtime.UseOrCreateObject()源码读者感兴趣可以自己看下
	obj, err := runtime.UseOrCreateObject(s.typer, s.creater, *actual, into)
	if err != nil {
		return nil, actual, err
	}

    // 反序列化对象，caseSensitiveJSONIterator暂且不用关心，此处可以理解为json.Unmarshal()。
    // 当然，读者非要知道个究竟，可以看看代码，笔者此处不注释。
	if err := caseSensitiveJSONIterator.Unmarshal(data, obj); err != nil {
		return nil, actual, err
	}

    // 如果是非strict模式，可以直接返回了。其实到此为止就可以了，后面是针对strict模式的代码，是否了解并不重要。
	if !s.options.Strict {
		return obj, actual, nil
	}

    // 笔者第一眼看到下面的以为看错了，但是擦了擦懵逼的双眼，发现就是YAMLToJSON。如果原始数据是json不会有问题么？
    // 笔者查看了一下yaml.YAMLToJSONStrict()函数注释：由于JSON是YAML的子集，因此通过此方法传递JSON应该是没有任何操作的。
    // 除非存在重复的字段，会解析出错。所以此处就是用来检测是否有重复字段的，当然，如果是yaml格式顺便转成了json。
    // 感兴趣的读者可以阅读源码，笔者只要知道它的功能就行了，就不“深究”了。
	altered, err := yaml.YAMLToJSONStrict(originalData)
	if err != nil {
		return nil, actual, runtime.NewStrictDecodingError(err.Error(), string(originalData))
	}
    // 接下来会因为未知的字段报错，比如对象未定义的字段，未打标签的字段等。
    // 此处使用DeepCopyObject()等同于新构造了一个对象，而这个对象其实又没什么用，仅作为一个临时的变量使用。
	strictObj := obj.DeepCopyObject()
	if err := strictCaseSensitiveJSONIterator.Unmarshal(altered, strictObj); err != nil {
		return nil, actual, runtime.NewStrictDecodingError(err.Error(), string(originalData))
	}
	// 返回反序列化的对象、GVK，所谓的strict模式无非是再做了一次转换和反序列化来校验数据的正确性，结果直接丢弃。
    // 所以说strict没有必要不用开启，除非你真正理解他的作用并且能够承受带来的后果。
	return obj, actual, nil
}

// gvkWithDefaults()利用defaultGVK补全actual中未设置的字段。
// 需要注意的是，参数'defaultGVK'只是一次调用相对于actual的默认GVK，不是Serializer.Decode()的默认GVK。
func gvkWithDefaults(actual, defaultGVK schema.GroupVersionKind) schema.GroupVersionKind {
    // actual如果没有设置Kind则用默认的Kind补全
	if len(actual.Kind) == 0 {
		actual.Kind = defaultGVK.Kind
	}
    // 如果Group和Version都没有设置，则用默认的Group和Version补全。
    // 为什么必须是都没有设置？缺少Version或者Group有什么问题么？下面的代码给出了答案。
	if len(actual.Version) == 0 && len(actual.Group) == 0 {
		actual.Group = defaultGVK.Group
		actual.Version = defaultGVK.Version
	}
    // 如果Version未设置，则用默认的Version补全，但是前提是Group与默认Group相同。
    // 因为Group不同的API即便Kind/Version相同可能是两个完全不同的类型，比如自定义资源(CRD)。
	if len(actual.Version) == 0 && actual.Group == defaultGVK.Group {
		actual.Version = defaultGVK.Version
	}
    // 如果Group未设置而Version与默认的Version相同，为什么不用默认的Group补全？
    // 前面已经解释过了，应该不用再重复了。
	return actual
}
```

### Encode

相比于解码，编码就简单很多，直接按照选项(yaml、pretty)编码就行了，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L297>

```go
// Encode()实现了Encoder.Encode()接口。
func (s *Serializer) Encode(obj runtime.Object, w io.Writer) error {
    // CacheableObject允许对象缓存其不同的序列化数据，以避免多次执行相同的序列化，这是一种出于效率考虑设计的类型。
    // 因为同一个对象可能会多次序列化json、yaml和protobuf，此时就需要根据编码器的标识符找到对应的序列化数据。
	if co, ok := obj.(runtime.CacheableObject); ok {
        // CacheableObject笔者不再注释了，感兴趣的读者可以自行阅读源码。
        // 其实根据传入的参数也能猜出来具体实现：利用标识符查一次map，如果有就输出，没有就调用一次s.doEncode()。
		return co.CacheEncode(s.Identifier(), s.doEncode, w)
	}
    // 非CacheableObject对象，就执行一次json的序列化。
	return s.doEncode(obj, w)
}

// doEncode()类似于于json.Marshal()，只是写入是io.Writer而不是[]byte。
func (s *Serializer) doEncode(obj runtime.Object, w io.Writer) error {
    // 序列化成yaml?
	if s.options.Yaml {
        // 序列化对象为json
		json, err := caseSensitiveJSONIterator.Marshal(obj)
		if err != nil {
			return err
		}
        // json->yaml
		data, err := yaml.JSONToYAML(json)
		if err != nil {
			return err
		}
        // 写入io.Writer。
		_, err = w.Write(data)
		return err
	}

    // 输出易于理解的格式？那么问题来了，为什么输出yaml不需要这个判断？很简单，yaml就是易于理解的。
	if s.options.Pretty {
        // 序列化对象为json
		data, err := caseSensitiveJSONIterator.MarshalIndent(obj, "", "  ")
		if err != nil {
			return err
		}
        // 写入io.Writer。
		_, err = w.Write(data)
		return err
	}

    // 非pretty模式，用我们最常用的json序列化方法，无非我们最常用方法是json.Marshal()。
    // 如果需要写入io.Writer，下面的代码是标准写法。
	encoder := json.NewEncoder(w)
	return encoder.Encode(obj)
}
```

### identifier

前面笔者提到了，编码器标识符可以看做是标签的字符串形式，大家应该很熟悉Kubernetes中的标签选择器，符合标签选择器匹配规则的所有API对象都可以看做"同质"。同样的道理，编码器也有自己的标签，标签相同的所有编码器是同质的，即编码同一个API对象的结果都是一样的。编码器标识符的定义没有那么复杂，就是简单的字符串，匹配也非常简单，标识符相等即为匹配，所以标识符可以理解为标签的字符串形式。源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L66>

```go
// identifier()根据给定的选项计算编码器的标识符。
func identifier(options SerializerOptions) runtime.Identifier {
    // 编码器的唯一标识符是一个map[string]string，不同属性的组合形成更了唯一性。
	result := map[string]string{
        // 名字是json，表明是json编码器。
		"name":   "json",
        // 输出格式为yaml或json
		"yaml":   strconv.FormatBool(options.Yaml),
        // 是否为pretty模式
		"pretty": strconv.FormatBool(options.Pretty),
	}
    // 序列化成json，生成最终的标识符，json序列化是标签的一种字符串形式。
	identifier, err := json.Marshal(result)
	if err != nil {
		klog.Fatalf("Failed marshaling identifier for json Serializer: %v", err)
	}
    // 也就是说，只要yaml和pretty选项相同的任意两个json.Serializer，任何时候编码同一个API对象输出一定是相同的。
    // 所以当API对象被多个编码器多次编码时，以编码器标识符为键利用缓冲避免重复编码。
	return runtime.Identifier(identifier)
}
```

## yaml

其实json.Serializer支持json和ymal，那么还有必要定义yaml.Serializer么？毕竟构造两个json.Serializer对象，一个开启yaml选型，一个关闭yaml选项就可以了。来看看yaml.Serializer的定义，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/yaml/yaml.go#L26>

```go
// yamlSerializer实现了runtime.Serializer()。
// 没有定义Serializer类型，而是一个包内的私有类型yamlSerializer，如果需要使用这个类型必须通过包内的公有接口创建。
type yamlSerializer struct {
    // 直接继承了runtime.Serializer，不用想肯定是json.Serializer。
	runtime.Serializer
}

// NewDecodingSerializer()向支持json的Serializer添加yaml解码支持。
// 也就是说yamlSerializer编码json，解码yaml，当然从接口名字看，调用这个接口估计只需要用解码能力吧。
// 好在笔者检索了一下源码，yamlSerializer以及NewDecodingSerializer()没有引用的地方，应该是历史遗留的代码。
func NewDecodingSerializer(jsonSerializer runtime.Serializer) runtime.Serializer {
	return &yamlSerializer{jsonSerializer}
}

// Decode()实现了Decoder.Decode()接口。
func (c yamlSerializer) Decode(data []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
    // yaml->json
	out, err := yaml.ToJSON(data)
	if err != nil {
		return nil, nil, err
	}
    // 反序列化json
	data = out
	return c.Serializer.Decode(data, gvk, into)
}
```

# 总结

1. json.Serializer可以实现json和yaml两种数据格式的序列化/反序列化，而yaml.Serializer基本不用了；
2. MetaFactory的功能就是提取apiVersion和kind字段，然后返回GVK；
3. json.Serializer也可以像json/yaml.Unmarshal()一样使用，只要传入的'into'的类型没有在Schema中注册就可以了；
4. json.Serializer在反序列化之前需要计算API对象的GVK，计算原则是优先使用json中的GVK，如果GVK有残缺，则采用默认GVK补全，最后用'into'指向的对象类型补全；
5. 其实runtime.Serializer只是比json/yaml.Unmarshal()多了类型提取并构造对象的过程，但是依然存在无法通用的问题，即解码json和yaml需要不同的对象，这就要[RecognizingDecoder](./RecognizingDecoder.md)来解决了；
