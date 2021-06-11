<!--
 * @Author: jinde.zgm
 * @Date: 2021-06-11 21:09:58
 * @Description: StreamSerializer解析
-->

# 前言

在[RecognizingDecoder](./RecognizingDecoder.md)文章中，笔者解析了具有辨识数据能力的解码器，并介绍了"万能解码器"，可以解码Kubernetes所有序列化格式（json、yaml、protobuf）的API对象。无论是runtime.Decoder还是[RecognizingDecoder](./RecognizingDecoder.md)，都是用于从内存([]byte)解码器单个API对象的，但是在Kubernetes的应用中，还有一种解码需求是从流中解码对象，比如比较常用的Watch功能，就是从流中持续读取数据并解码成一个一个的API对象。

从数据流中读取数据并解码API对象的解码器本文命名为**流解码器**，我们日常说的流是字节流，那么流解码器可以看做是API对象流，每调用一次返回一个API对象。如果想把字节流变成API对象流，首先需要将字节流变成json/yaml/protobuf序列化格式对象流，即先从字节流中读取一个一个的json/yaml/protobuf对象。然后再用解码器将json/yaml/protobuf对象解码成API对象就完成了。

# StreamSerializer

## StreamSerializerInfo

既然Kubernetes有基于流读/写API对象的需求，StreamSerializerInfo为实现这个需求提供了支撑，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/interfaces.go#L134>

```go
// StreamSerializerInfo含有指定序列化格式流的序列化器(Serializer)信息，这句话是不是拗口，可以总结为一下2点：
// 1. 指定的序列化格式，json/yaml/protubuf之一；
// 2. 只包含流序列化器的信息，但不是流序列化器，说的直白点就是基于StreamSerializerInfo可以构造流序列化器
type StreamSerializerInfo struct {
    // 标识此序列化器是否可以编码为UTF-8，举个例子：json.Serializer就可以编码UTF8，而protobuf.Serializer就不可以。
	EncodesAsText bool
    // 序列化器对象，可能是json.Serializer或protobuf.Serializer。
    // 关于Serializer参看 https://github.com/jindezgm/k8s-src-analysis/blob/master/apimachinery/runtime/Serializer.md
	Serializer
    // 在介绍Framer之前，笔者先介绍一下帧的概念。帧这个概念在很多地方都有应用，比如视频、网络通信。
    // 视频是由连续的图片组成，每一个图片就是一帧；而网络通信中的帧是数据链路层的协议数据单元。
    // 我们发现帧的概念一般应用在连续数据，可以将连续的数据分成以帧为单位的单元，此处的帧也是相同的原理。
    // 本文中流是API对象序列化数据的字节流，而帧就是该字节流的数据元，以json为例一帧就是一个json对象。
    // 按照这个思路在来看Framer就非常容易理解了，Framer是工厂类，该工厂可以构建读/写帧的对象。
    // 关于Framer参看下面的源码注释。
	Framer
}

// Framer是工厂类，用于构建帧的Reader和Writer
type Framer interface {
    // 构建帧Reader，该Reader每次读取一个API的序列化格式对象，比如读取一个json对象。
	NewFrameReader(r io.ReadCloser) io.ReadCloser
    // 构建帧Writer，该Writer每次写入一个API的序列化格式对象，比如写入一个json对象。
	NewFrameWriter(w io.Writer) io.Writer
}
```

既然Framer是一个interface，自然就要有实现这个接口的类型。因为从字节流中提取不同序列化格式的帧的方法不同，所以Kubernetes为每种序列化格式都实现了Framer接口，本文以json为例做代码解析，yaml和protobuf建议读者自己查看源码。

### json

因为json有明显的特征"{}"，所以找到成对的"{}"即为一个json对象，我们来看看Kubernetes的实现方法是不是跟笔者的想法一样。源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/json/json.go#L347>

```go
// jsonFramer实现了runtime.Framer。
type jsonFramer struct{}

// NewFrameWriter()实现了runtime.Framer.NewFrameWriter().
func (jsonFramer) NewFrameWriter(w io.Writer) io.Writer {
    // 其实写并没有什么特殊的实现，只要一个json接一个json写就行了，所以原生的io.Writer就能满足要求。
	return w
}

// NewFrameReader()实现了runtime.Framer.NewFrameReader().
func (jsonFramer) NewFrameReader(r io.ReadCloser) io.ReadCloser {
    // 通过另一个工具报实现，下面有注释
	return framer.NewJSONFramedReader(r)
}
```

正是因为写json不需要特殊实现，而读json需要根据json的数据格式特点读取数据，所以将json帧Reader单独封装在工具包中，源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/util/framer/framer.go#L122>

```go
// jsonFrameReader实现了io.ReadCloser，从io.ReadCloser中以json对象为粒度读取数据。
// 这有点类似于jave的各种InputStream的套接。
type jsonFrameReader struct {
    // jsonFrameReader从r一个接一个读取json对象
	r         io.ReadCloser
    // 用于读取json对象，需要注意的是此处json包就是我们常用的json.Marshal()的那个包。
    // 因为json.Decoder.Decode()具备从io.Reader读取一个json对象的能力，所以jsonFrameReader复用了这部分能力。
    // 但是json.Decoder.Decode()读取一个json对象后会执行反序列化操作，而jsonFrameReader根本不需要反序列化操作。
    // jsonFrameReader是如何利用json.Decoder实现只读取json对象而不执行序列化操作的呢？详情见后面的实现。
	decoder   *json.Decoder
    // remaining这个变量的存在是因为json对象大小不一致造成的，因为每个API对象的大小不可能都相同。
    // 用户调用Read()接口的时候无法预知json的大小，提供的缓冲可能不足以容纳一个json对象。
    // 此时部分json存入用户的缓存中，而剩余的部分存在remaining中，此时Read()会返回io.ErrShortBuffer。
	remaining []byte
}

// NewJSONFramedReader()是jsonFrameReader的构造函数，更准确的说是json的io.ReadCloser构造函数。
func NewJSONFramedReader(r io.ReadCloser) io.ReadCloser {
	return &jsonFrameReader{
		r:       r,
        // 构造json.Decoder
		decoder: json.NewDecoder(r),
	}
}

// Read()实现了io.ReadCloser.Read()接口，用于读取一个json对象到'data'中。
// 如果len(data)小于json对象的大小，则'data'中存储部分json，并返回io.ErrShortBuffer错误。
func (r *jsonFrameReader) Read(data []byte) (int, error) {
    // 如果remaining有残留的数据，说明上次调用Read()并没有读取完整，需要将剩余的部分数据继续输出到'data'中。
	if n := len(r.remaining); n > 0 {
        // 如果'data'的大小比剩余的数据量大，直接将剩余的数据拷贝到'data'中。
		if n <= len(data) {
            // 下面的代码还是比较讲究的，必须需要解释一下：
            // 1. data[0:0]将len(data)变成0，但是内存的地址不变；
            // 2. append()用于将remain中的数据拷贝到data中，并将data的大小修改为len(r.remaining)
            // 其实下面的代码也可以写成这样: data = data[:copy(data, r.remain)]
			data = append(data[0:0], r.remaining...)
			r.remaining = nil
			return n, nil
		}

        // 如果data的大小依然比remain小，那么继续将部分数据拷贝到data中并返回io.ErrShortBuffer
		n = len(data)
		data = append(data[0:0], r.remaining[:n]...)
		r.remaining = r.remaining[n:]
		return n, io.ErrShortBuffer
	}

    // 获取data的大小，因为下面需要临时将data的大小调整为0，此处需要记录一下。
	n := len(data)
    // json.RawMessage这就是笔者前面提到利用json.Decoder只读取不反序列化的关键。
    // json.RawMessage是[]byte类型的重定义，所以可以将data转换为json.RawMessage类型。
    // json.RawMessage实现了Unmarshal接口，所以json.Decoder会调用json.RawMessage.Unmarshal()反序列化对象。
    // json.RawMessage.Unmarshal()的实现就是简单的拷贝数据，所以看似调用了Decode()实则做了拷贝。
    // 具体的实现读者可以阅读json.Decoder.Decode()的源码实现，笔者点到为止即可。
	m := json.RawMessage(data[:0])
	if err := r.decoder.Decode(&m); err != nil {
		return 0, err
	}

    // 这里的实现也非常精髓，笔者需要解释一下:
    // 1. 首先通过类型转换的方式将data[:0]赋值给m，这就是精髓的所在，m地址指向了data，但是大小为0；
    // 2. json.RawMessage.Unmarshal()通过append()拷贝读取到的json对象，那么就会出现两种情况：
    //    2.1. cap(data)大于等于读取的json数据大小，那么json数据会直接拷贝到data中;
    //    2.2. cap(data)小于读取的json数据大小，则m会被赋值新缓冲，并且将数据拷贝到新的缓冲中；
    // 3. 简单直白的说：如果data的大小足以容纳读取的json对象则直接读取到data中，否则读取到新申请的缓存中；
	if len(m) > n {
        // len(m) > n说明m指向了新缓存，那么需要将一部分数据拷贝到data中，再用remain指向剩余部分数据的地址
		data = append(data[0:0], m[:n]...)
		r.remaining = m[n:]
		return n, io.ErrShortBuffer
	}
	return len(m), nil
}
```

虽然函数代码不多，逻辑也不复杂，但是实现的细节还是满满的，尽量避免不必要的内存拷贝，这需要对golagn的slice类型有充分的认识才行。

# streaming

StreamSerializerInfo只是流序列化器的信息，没有任何序列化和反序列化能力。也就是说StreamSerializerInfo提供了构造流序列化器的参数（Framer和Serializer），但不是流序列化器。Kubernetes在streaming包中定义了流解码器和流编码器，专门基于流解码/编码API对象，本章节将解析流解码器/编码器的实现。

需要注意，笔者为了简便，将streaming.Decoder和streaming.Encoder简写成Decoder和Encoder；为了与runtime.Decoder和runtime.Encoder有所区分，runtime.Decoder和runtime.Encoder依然采用全名。

## Decoder

Decoder也是一种解码器，与runtime.Decoder就好像json.Decoder.Decode()与json.Unmarshal()之间的区别，前者基于流解码API对象，后者基于[]byte解码API对象。源码链接：<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/streaming/streaming.go#L38>

```go
// Decoder定义了流解码器的接口
type Decoder interface {
    // 从流中解码一个API对象，当没有更多的对象时，将返回io.EOF。
    // 相比于runtime.Decoder，参数只是少了输入的数据([]byte)，其他部分(参数、返回值)都是一样的。
    // 这个非常好理解，因为数据来源于流，需要Decoder读取流中的数据并解码对象。
	Decode(defaults *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error)
    // 关闭流解码器，其实主要是为了关闭底层的流（io.ReadCloser）
	Close() error
}

// decoder实现了Decoder
type decoder struct {
    // 输入数据流，其实就是Framer.NewFrameReader()构造的序列化格式的对象流，比如json流。
	reader    io.ReadCloser
    // 解码器，用于解码API对象，decoder应该与reader匹配，即都是json、yaml或者protobuf。
    // 这就是StreamSerializerInfo存在的价值，它提供了指定格式的Framer和Serializer。
	decoder   runtime.Decoder
    // 数据缓冲，用于缓冲从reader读取的数据，然后在调用decoder来解码
	buf       []byte
    // 缓冲最大值，单位为字节
	maxBytes  int
    // 从变量名字来看是复位读取的标记，但是什么是复位读取，为什么复位读取？
    // 笔者先提一个问题，如果一个序列化对象非常大，以至于最大缓冲都无法容纳该怎么办？笔者认为：
    // 1. 这个对象肯定是无法解码的，因为缓冲无法容纳这个对象，也就无法调用解码器解码，所以只能返回一个错误表示对象太大；
    // 2. 这个超大的对象不能影响后续对象(因为是流)的解码；
    // 基于以上的分析，decoder在遇到一个超大对象后会返回一个错误，但是再次调用解码的时候流中可能残留超大对象的部分数据。
    // 所以resetRead的就是用来标记上一次是否读到了超大对象，当成功读取一个序列化对象且resetRead=true就是上一个超大对象的结尾，
    // 所以需要在复位读取一次，详情参看Decode()接口实现。
	resetRead bool
}

// NewDecoder()用于创建一个流解码器，从'r'中读取序列化对象并用'd'解码对象。
// 此处对'r'有一些要求，当读取数据时提供的缓冲不够读取一个序列化对象的数据时需要返回ErrShortRead错误。
// 需要注意的是，此处的io.ReadCloser其实是由Framer.NewFrameReader()构建的。
func NewDecoder(r io.ReadCloser, d runtime.Decoder) Decoder {
	return &decoder{
		reader:   r,
		decoder:  d,
		buf:      make([]byte, 1024),   // 初始申请1024大小的缓冲
		maxBytes: 16 * 1024 * 1024,     // 缓冲最大为16MB。
	}
}

// Decode()实现了Decoder.Decode()。
func (d *decoder) Decode(defaults *schema.GroupVersionKind, into runtime.Object) (runtime.Object, *schema.GroupVersionKind, error) {
    // base是一个"指针"，指向了未来从流中读取数据到缓冲的起始位置，初始为0就是读取的数据放到缓冲的起始位置。
    // 这么设计的目的是什么？很简单，序列化对象大小可能比当前缓冲大，一次读取不完整，缓冲扩容后再次读取时，
    // 指针就要指向缓冲已经读取对象数据的下一个位置，这个设计很简单，没什么难度。
	base := 0
    // 为什么需要无限循环，因为可能一次读不完，如果缓冲太小以至于无法读取一个完整的序列化对象，就需要持续读取直到读完为止。
	for {
        // 尝试读取一个序列化对象(帧)
		n, err := d.reader.Read(d.buf[base:])
        // 缓冲大小不足？
		if err == io.ErrShortBuffer {
            // 读取了0个字节，又返回缓冲大小不足，这应该是异常了吧？
			if n == 0 {
				return nil, nil, fmt.Errorf("got short buffer with n=0, base=%d, cap=%d", base, cap(d.buf))
			}
			if d.resetRead {
                // d.resetRead为true表示上一次读取到了超大对象，此次读取的数据还是该超大对象的一部分，
                // 所以应该丢弃当前读取的数据并继续读取直到读取到结尾为止。resetRead为true说明缓冲已经达到最大了，所以也没必要扩容了。
				continue
			}
            // 如果缓冲大小还没有达到最大值，就扩容缓冲，再读一次
			if len(d.buf) < d.maxBytes {
                // 偏移指针到下一次读取数据存放的位置
				base += n
                // 在原有缓冲大小的基础上再追加同等大小的空数据，可以理解为缓冲大小扩容2倍。
                // 一旦扩容后的大小大于cap(d.buf)就会申请新的内存，需要注意这个操作：拷贝已经读取的数据到新的缓冲中
				d.buf = append(d.buf, make([]byte, len(d.buf))...)
				continue
			}
            // 对象太大了，返回ErrObjectTooLarge的同时将resetRead设置为true， 
            // 标识下次调用的时候需要将该超大对象的残留数据读取出来并丢弃。
			d.resetRead = true
			base = 0 // 其实这个赋值没啥必要，毕竟需要退出函数了
			return nil, nil, ErrObjectTooLarge
		}
        // 其他错误直接返回，大部分应该是io.EOF，当然也有其他错误的可能。
		if err != nil {
			return nil, nil, err
		}
        // 到此，已经读取了一个完整的对象数据
		if d.resetRead {
            // resetRead为true说明上一次解码读取了一个超大的对象，当前读取完该超大对象的残留数据。
            // 清除resetRead标记，重新读取一次才是此次需要解码的API对象，这也就是复位的由来吧。
			d.resetRead = false
			continue
		}
        // 偏移指针，因为要读取循环，一次指针偏移的目的就是为了计算读取的数据大小
		base += n
		break
	}
    // 解码对象，因为序列化对象的最后一个字节在d.buf[base-1]，所以用缓冲的[0,base)来解码API对象。
	return d.decoder.Decode(d.buf[:base], defaults, into)
}
```

## Encoder

既然有流解码的需求，流编码器也是有存在的必要的。流解码器作为消费者消费流中的对象，而流编码器则是作为生产者向流中写入对象。流编码器要简单很多，只需要一个接一个对象的编码并写入流即可，<https://github.com/kubernetes/apimachinery/blob/release-1.21/pkg/runtime/serializer/streaming/streaming.go#L31>

```go
// Encoder定义了流编码器的接口。
type Encoder interface {
    // 将obj写入流，与runtime.Encoder.Encode()接口不同的是参数中没有io.Writer参数。
    // 因为在构造Encoder对象时提供了io.Writer，这样就不用每次调用Encode()时再提供了。
	Encode(obj runtime.Object) error
}

// encoder实现了Encoder接口。
type encoder struct {
    // 这个应该不用介绍了
	writer  io.Writer
    // 因为Encode传入的是API对象，需要将API对象编码成字节数据后在写入流
	encoder runtime.Encoder
    // 用来缓存API对象编码后的数据。
    // 因为bytes.Buffer实现了io.Writer，所以runtime.Encoder.Encode()可以将序列化的数据写入bytes.Buffer。
	buf     *bytes.Buffer
}

// NewEncoder()是Encoder的构造函数，需要提供io.Writer和解码器，这个应该比较好理解。
func NewEncoder(w io.Writer, e runtime.Encoder) Encoder {
	return &encoder{
		writer:  w,
		encoder: e,
		buf:     &bytes.Buffer{},
	}
}

// Encode()实现了Encoder.Encode()。
func (e *encoder) Encode(obj runtime.Object) error {
    // 编码API对象。
	if err := e.encoder.Encode(obj, e.buf); err != nil {
		return err
	}
    // 将序列化数据写入流
	_, err := e.writer.Write(e.buf.Bytes())
    // 复位缓存，为下次编码做准备
	e.buf.Reset()
	return err
}
```

# 总结

1. streaming.Decoder和runtime.Decoder都是用来解码器API对象，前者基于流(io.Reader)解码，后者基于缓存([]byte)解码;
2. streaming.Encoder和runtime.Encoder都是用来编码器API对象，前者基于流(io.Reader)编码，后者基于缓存([]byte)编码;
3. 流编解码器的典型应用场景是[RSETClient的Watch](../client-go/rest/../../../client-go/rest/Request.md#Watch)
