<!--
 * @Author: jinde.zgm
 * @Date: 2021-06-03 21:48:34
 * @Description: RecognizingDecoder解析
-->

# 前言

在[Serializer](./Serializer.md)文章中提到了，Serializer虽然抽象了序列化/反序列化的接口，但是反序列化不同格式的数据需要不同的Serializer对象，并不是很通用。有没有一种类型可以反序列化任意格式的数据呢？答案是有的，但是在介绍这个“万能解码器”之前，需要引入RecognizingDecoder接口。

本文引用源码为kubernetes的release-1.21分支。

# RecognizingDecoder

如果笔者自己实现“万能解码器”，我抽象一个接口用于判断解码器是否能够解码当前格式的数据。让解码器实现这个接口，因为只有解码器自己才了解序列化数据的"特征"（早年笔者做视频存储产品时就是这么做的）。Kubernetes将这个判断过程称之为Recognize，笔者翻译为辨识。那么实现了辨识接口的解码器就是RecognizingDecoder，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/recognizer/recognizer.go#L26>

```go
// RecognizingDecoder是具有辨识能力的解码器。
type RecognizingDecoder interface {
    // 继承了解码器，很好理解，因为本身就是解码器。
	runtime.Decoder
    // 从序列化数据中提取一定量(也可以是全量)的数据(peek)，根据这部分数据判断序列化数据是否属于该解码器。
    // 举个栗子，如果是json解码器，RecognizesData()就是判断peek是不是json格式数据。
    // 如果辨识成功返回ok为true，如果提供的数据不足以做出决定则返回unknown为true。
    // 首先需要注意的是，在1.20版本，该接口的定义为RecognizesData(peek io.Reader) (ok, unknown bool, err error)。
    // 因为io.Reader.Read()可能会返回错误，导致该接口也可能返回错误，到了1.21版本改成[]byte就不会返回错误了。
    // 所以该接口返回的可能性如下：
    // 1. ok=false, unknown=true: peek提供的数据不足以做出决定，什么情况会返回unknown后续章节会给出例子
    // 2. ok=true, unknown=false: 辨识成功，序列化数据属于该解码器，比如json解码器辨识json数据
    // 3. ok&unknown=false: 辨识失败，序列化数据不属于该解码器，比如json解码器辨识yaml或protobuf数据
	RecognizesData(peek []byte) (ok, unknown bool, err error)
}
```

在[Serializer](./Serializer.md#json)的文章中介绍了json.Serializer，它是json和yaml的编解码器(runtime.Serializer)，自然它就是json和yaml的解码器。但是[Serializer](./Serializer.md#json)的文章中笔者仅对runtime.Serializer相关的功能做了解析，其实它也实现了RecognizingDecoder。接下来的章节将解析json、yaml、protobuf解码器是如何实现辨识数据接口(RecognizingDecoder.RecognizesData())。

## json&yaml

在[Serializer](./Serializer.md#json)的文章中已经介绍了json.Serializer是json和yaml的解码器，所以json.Serializer可以辨识json和yaml两种格式的数据，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L336>

```go
// RecognizesData()实现了RecognizingDecoder.RecognizesData()接口
func (s *Serializer) RecognizesData(data []byte) (ok, unknown bool, err error) {
    // 如果是yaml选项(即yaml编解码器)，直接返回unknown，这个操作可以啊，不打算争取一下了么？
    // 其实道理很简单，yaml实在是没有什么明显的特征，所以返回unknown，表示不知道是不是yaml格式。
	if s.options.Yaml {
		return false, true, nil
	}
    // 既然无法辨识yaml格式，那json格式总不至于也无法辨识吧，毕竟json有明显的特征"{}"。
    // utilyaml.IsJSONBuffer()的函数实现就是找数据中是否以'{'开头(前面的空格除外)。
    // 这个原理就很简单，因为API类型都是结构体（不存在数组和空指针），所以json数据都是'{...}'格式。
	return utilyaml.IsJSONBuffer(data), false, nil
}
```

本以为辨识数据格式有多么高深？其实就是很简单的事情，找到数据格式的明显特征就可以了。只要这个特征在所有的数据格式中是独一无二的，越简单越好。如果辨识数据非常复杂，甚至与解码一样的计算量，辨识就没有意义了，毕竟设计辨识就是为了避免无意义的解码。

## protobuf

因为json和yaml是开放的、标准的数据格式，它们只定义了数据的格式。而protobuf是一种协议专属的数据格式，所以一般会在数据头增加一些关键字，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/protobuf/protobuf.go#L248>

```go
// RecognizesData()实现了RecognizingDecoder.RecognizesData()接口.
func (s *Serializer) RecognizesData(data []byte) (bool, bool, error) {
    // 只要数据是以protobuf协议关键字开始就是protobuf序列化数据，protobuf关键字是[]byte{0x6b, 0x38, 0x73, 0x00}。
	return bytes.HasPrefix(data, s.prefix), false, nil
}
```

# decoder

虽然json，yaml，protobuf的解码器都实现了RecognizingDecoder，但是他们依然是一个个分散的解码器，是不是应该有一个类型将这些解码器管理器来，然后可以做到自动辨识并解码的能力？它就是万能解码器decoder，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/recognizer/recognizer.go#L51>

```go
// decoder实现了RecognizingDecoder。
type decoder struct {
    // 管理了若干解码器，为什么是[]runtime.Decoder，而不是[]RecognizingDecoder。
    // json，yaml，protobuf的解码器不都实现了RecognizingDecoder么？
    // 笔者猜测这是为扩展考虑的，未来如果增加一种解码器，但是又没有实现辨识接口怎么办？
    // 那么问题来，没有实现辨识接口的解码器还有必要被decoder管理么？
    // 答案是有必要，在非常极端的情况下decoder会尝试让所有解码器都解码一次，只要有一个成功就行，后面会看到代码实现。
	decoders []runtime.Decoder
}

// decoder毕竟是包内的私有类型，如果需要使用它必须通过包内的公有函数构造。
// NewDecoder()是decoder的构造函数，传入各种数据格式的解码器。
func NewDecoder(decoders ...runtime.Decoder) runtime.Decoder {
	return &decoder{
		decoders: decoders,
	}
}
```

## RecognizesData

其实decoder没有必要实现RecognizesData()接口，因为NewDecoder()返回的是runtime.Decoder，只要实现Decode()接口即可。既然是在recognizer包内定义的类型，既来之则安之吧，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/recognizer/recognizer.go#L57>

```go
// RecognizesData()实现了RecognizingDecoder.RecognizesData()接口。
// 因为decoder结构体中包含了[]runtime.Decoder成员变量(decoders)，为了区分decoder和decoder.decoders[i]，
// 后续的注释中'decoder'代表的就是decoder结构体，而'解码器'代表的是decoder.decoders[i]。
func (d *decoder) RecognizesData(data []byte) (bool, bool, error) {
	var (
		lastErr    error
		anyUnknown bool
	)
    // 遍历所有的解码器
	for _, r := range d.decoders {
        // 将解码器划分为[]RecognizingDecoder和[]runtime.Decoder两个子集，其中只有[]RecognizingDecoder具备辨识能力。
		switch t := r.(type) {
        // 只用decoder.decoders中的[]RecognizingDecoder子集辨识数据格式
		case RecognizingDecoder:
            // 让每个解码器都辨识一下序列化数据
			ok, unknown, err := t.RecognizesData(data)
			if err != nil {
                // 如果解码器辨识出错，则只需要记录最后一个错误就可以了，因为所有解码器报错都应该是一样的。
                // 前面已经提到了，因为历史原因，io.Reader读取数据可能返回错误，当前版本已经不会返回错误了。
				lastErr = err
				continue
			}
            // 只要有任何解码器返回数据不足以做出决定，那么就要返回unknown，除非有任何解码器辨识成功。
            // 这个逻辑应该比较简单：当有一些解码器返回unknown，其他都返回无法辨识的时候，对于decoder而言就是返回unknown。
            // 因为有些解码器只是不知道数据格式是不是属于自己，并不等于辨识失败，是存在可能的。
			anyUnknown = anyUnknown || unknown
			if !ok {
				continue
			}
            // 只要有任何一个解码器返回辨识成功，那么就可以直接返辨识成功了。
			return true, false, nil
		}
	}
    // 所有的解码器都没有辨识成功，那么如果有任何解码器返回unknown就返回unknown。
	return false, anyUnknown, lastErr
}
```

## Decode

Decode()才是decoder的精髓所在，因为decoder是万能解码器，不"挑选"数据格式，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/recognizer/recognizer.go#L80>

```go
// Decode()实现了runtime.Decoder.Decode()接口。
func (d *decoder) Decode(data []byte, gvk *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
	var (
		lastErr error
		skipped []runtime.Decoder
	)

    // 遍历所有的解码器
	for _, r := range d.decoders {
		switch t := r.(type) {
        // 找到[]RecognizingDecoder子集，与RecognizesData()类似。
		case RecognizingDecoder:
            // 这个过程在decoder.RecognizesData()已经注释过了，功能是一样的。
			ok, unknown, err := t.RecognizesData(data)
			if err != nil {
				lastErr = err
				continue
			}
            // 把返回unknown的解码器都汇总到skipped中，表示这些先略过，因为他们不知道数据格式是否属于自己。
			if unknown {
				skipped = append(skipped, t)
				continue
			}
            // 解码器明确返回数据不属于解码器，辨识失败，忽略该解码器。需要注意是“忽略”，不是“略过”。
            // “略过”的解码器未来可能还会有用，而忽略的解码器是不会再用的。
			if !ok {
				continue
			}
            // 如果解码器辨识成功，则用该解码器解码
			return r.Decode(data, gvk, into)
        // []runtime.Decoder子集也放入skipped中，他们不具备辨识数据的能力，与unknown本质是一样的。
		default:
			skipped = append(skipped, t)
		}
	}

	// 如果没有任何解码器辨识数据成功，那就用[]runtime.Decoder子集和返回unknown的[]RecognizingDecoder子集逐一解码。
    // 这是一个非常简单暴力的做法，但是也没什么好办法。但是不用过分担心，绝大部分情况是可辨识成功的。
    // 也就是说，只有非常极端的情况才会执行这里的代码。
	for _, r := range skipped {
		out, actual, err := r.Decode(data, gvk, into)
		if err != nil {
			lastErr = err
			continue
		}
		return out, actual, nil
	}

	if lastErr == nil {
		lastErr = fmt.Errorf("no serialization format matched the provided data")
	}
	return nil, nil, lastErr
}
```

既然decoder的两个接口都需要明确区分`[]RecognizingDecoder`和`[]runtime.Decoder`两个子集，笔者打算提交一个PR，将decoder管理的解码器划分为两个子集，在NewDecoder()构造decoder时将所有解码器划分到两个自己，这样就不用每次在解码的时候再做类型断言了。

# 总结

1. RecognizingDecoder是具有数据辨识能力的解码器，它继承了runtime.Decoder并增加了RecognizesData()接口。
2. RecognizingDecoder只是比runtime.Decoder增加辨识数据的接口，但是不代表RecognizingDecoder.Decode()会自动辨识数据并解码。举个例子：json.Serializer实现了RecognizingDecoder，但是只能解码json或yaml的数据。
3. recognizer.decoder能够自动辨识数据格式并用相应的解码器解码。
4. recognizer.decoder辨识的方法是：如果有解码器辨识成功就用该解码器解码，否则就用辨识失败以外的解码器逐一尝试解码。
5. 虽然感觉辨识是一个尝试行为，其实是一个非常靠谱的动作，因为Kubernetes只有3中数据格式，json，yaml和protobuf，json和protobuf可以明确的返回是否辨识成功，不存在unkonwn的可能。虽然yaml无法辨识，但是此时已经明确排除了json和protobuf，也就只剩下yaml一种可能了。
6. recognizer.decoder到底有什么用？[CodecFactory](CodecFactory.md).UniversalDeserializer()返回的解码器就是recognizer.decoder，而这个接口官方注释是："能够解码所有已知格式的API对象"。与笔者叫的“万能解码器”是一个意思。
