本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

### 简介

不管什么编程语言涉及到跨进程通信有不同的通信协议，如熟知的网络通信协议HTTP即超文本协议传输协议明文时传输，再比如rpc熟知的dubbo协议(可见我本git整理)；如spring-clound fegin就是采用HTTP协议；再比如我们自己可以使用netty实现各种通信协议；也可以自己基于内核底层socket通信；

为什么会出现不同的协议？

他们有自己的历史背景，也出现标准协议，也有协议被某些编程语言采用，也有些协议可以跨多语言通信；

如果我们研究这些通信协议，那么选择哪个协议比较好入门，对于后端来说，rpc与grpc会经常碰到过，如dubbo、grpc的protobuf协议；

接下来我会使用golang语言进行详细解析下；



### Protobuf如何使用

研究一门技术，需要先入门，不入门谈技术就像纸上谈兵，没亲身入门没亲身经历过，泛泛之谈也可增加一些视野；

#### 概念

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

protobuf协议是以一个 .proto 后缀的文件为基础，这个文件描述了存在哪些数据，数据类型是怎么样的。

protocol buffers 是一种语言无关、平台无关、可扩展的序列化结构数据的方法，它可用于（数据）通信协议、数据存储等。

Protocol Buffers 是一种灵活，高效，自动化机制的结构数据序列化方法－可类比 XML，但是比 XML 更小（3 ~ 10倍）、更快（20 ~ 100倍）、更为简单。

你可以定义数据的结构，然后使用特殊生成的源代码轻松的在各种数据流中使用各种语言进行编写和读取结构数据。你甚至可以更新数据结构，而不破坏由旧数据结构编译的已部署程序。



#### 定义protobuf

**一些说明**

- required : 不可以增加或删除的字段，必须初始化；
- optional :  可选字段，可删除，可以不初始化；
- repeated : 可重复字段， 对应到java文件里，生成的是List；
- 定义一种结构体必须是message修饰，允许内嵌其他自定义的message作为自己的属性；
- 每个属性有自己的编号，这个编号不能随便修改；
- 每个属性都有自己的标量值类型，编译成每种语言的修饰符不一样；



**标量值类型**

标量 message 字段可以具有以下几种类型之一 - 该表显示 .proto 文件中指定的类型，以及自动生成的类中的相应类型：

| .proto Type |                            Notes                             | C++ Type | Java Type  | Python Type[2] | Go Type  |
| :---------: | :----------------------------------------------------------: | :------: | :--------: | :------------: | :------: |
|   double    |                                                              |  double  |   double   |     float      | *float64 |
|    float    |                                                              |  float   |   float    |     float      | *float32 |
|    int32    | 使用可变长度编码。编码负数的效率低 - 如果你的字段可能有负值，请改用 sint32 |  int32   |    int     |      int       |  *int32  |
|    int64    | 使用可变长度编码。编码负数的效率低 - 如果你的字段可能有负值，请改用 sint64 |  int64   |    long    |  int/long[3]   |  *int64  |
|   uint32    |                       使用可变长度编码                       |  uint32  |   int[1]   |  int/long[3]   | *uint32  |
|   uint64    |                       使用可变长度编码                       |  uint64  |  long[1]   |  int/long[3]   | *uint64  |
|   sint32    | 使用可变长度编码。有符号的 int 值。这些比常规 int32 对负数能更有效地编码 |  int32   |    int     |      int       |  *int32  |
|   sint64    | 使用可变长度编码。有符号的 int 值。这些比常规 int64 对负数能更有效地编码 |  int64   |    long    |  int/long[3]   |  *int64  |
|   fixed32   |    总是四个字节。如果值通常大于 228，则比 uint32 更有效。    |  uint32  |   int[1]   |  int/long[3]   | *uint32  |
|   fixed64   |    总是八个字节。如果值通常大于 256，则比 uint64 更有效。    |  uint64  |  long[1]   |  int/long[3]   | *uint64  |
|  sfixed32   |                         总是四个字节                         |  int32   |    int     |      int       |  *int32  |
|  sfixed64   |                         总是八个字节                         |  int64   |    long    |  int/long[3]   |  *int64  |
|    bool     |                                                              |   bool   |  boolean   |      bool      |  *bool   |
|   string    |       字符串必须始终包含 UTF-8 编码或 7 位 ASCII 文本        |  string  |   String   | str/unicode[4] | *string  |
|    bytes    |                     可以包含任意字节序列                     |  string  | ByteString |      str       |  []byte  |

在 [Protocol Buffer 编码](https://www.jianshu.com/p/82ff31c6adc6) 中你可以找到有关序列化 message 时这些类型如何被编码的详细信息。

[1] 在 Java 中，无符号的 32 位和 64 位整数使用它们对应的带符号表示，第一个 bit 位只是简单的存储在符号位中。
 [2] 在所有情况下，设置字段的值将执行类型检查以确保其有效。
 [3] 64 位或无符号 32 位整数在解码时始终表示为 long，但如果在设置字段时给出 int，则可以为int。在所有情况下，该值必须适合设置时的类型。见 [2]。
 [4] Python 字符串在解码时表示为 unicode，但如果给出了 ASCII 字符串，则可以是 str（这条可能会发生变化）。



**定义protobuf通信结构**

```protobuf
message SearchResponse {
  repeated Result result = 1;
}

message Result {
  required string url = 1;
  optional string title = 2;
  repeated string snippets = 3;
}
```

**定义protobuf服务**

如果要将 message 类型与 RPC（远程过程调用）系统一起使用，则可以在 .proto 文件中定义 RPC 服务接口，protocol buffer 编译器将使用你选择的语言生成服务接口代码和存根。因此，例如，如果要定义一个 RPC 服务，其中具有一个获取 SearchRequest 并返回 SearchResponse 的方法，可以在 .proto 文件中定义它，如下所示：

```protobuf
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

**完整的protobuf文件**

```protobuf
syntax = "proto3"; //指定proto版本
package test;

message SearchRequest {
    UrlVO url = 1;
    string bizType = 13;
    int32 runMode = 14;
}

message UrlVO {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
}

message SearchResponse {
    string requestID = 1;
    bool success = 2;
    string message = 3;
    repeated string jobs = 4;
}

service SearchService {
    rpc Search (SearchRequest) returns (SearchResponse) {
    }
}
```



#### **编译protobuf**



**你的 .proto 文件将生成什么？**

当你在 `.proto` 上运行 protocol buffer 编译器时，编译器将会生成所需语言的代码，这些代码可以操作文件中描述的 message 类型，包括获取和设置字段值、将 message 序列化为输出流、以及从输入流中解析出 message。

- 对于 **C++**，编译器从每个 .proto 生成一个 .h 和 .cc 文件，其中包含文件中描述的每种 message 类型对应的类。
- 对于 **Java**，编译器为每个 message 类型生成一个 .java 文件（类），以及用于创建 message 类实例的特殊 Builder 类。
- **Python** 有点不同 -  Python 编译器生成一个模块，其中包含 .proto 中每种 message 类型的静态描述符，然后与元类一起使用以创建必要的 Python 数据访问类。
- 对于 **Go**，编译器会生成一个 .pb.go 文件，其中包含对应每种 message 类型的类型。
   你可以按照所选语言的教程了解更多有关各种语言使用 API ​​的信息。有关更多 API 详细信息，请参阅相关的 [API 参考](https://developers.google.com/protocol-buffers/docs/reference/overview)。

> ```
> protoc安装
> go get -u github.com/golang/protobuf/protoc-gen-go
> protoc --go_out=plugins=grpc:. search.proto
> ```

最后你会发现会生成一个 `search.pb.go`文件，需要把它移到对应自己的package下；

你可以重点观察下这个文件对应的通信结构以及生成对应通信的invoke

```go
func (c *searchServiceClient) Search(ctx context.Context, in *SearchRequest, opts ...grpc.CallOption) (*SearchResponse, error) {
	out := new(SearchResponse)
	err := c.cc.Invoke(ctx, "/test.SearchService/Search", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

对于如何整合grpc与etcd进行provider与consumer之间交互，可以翻下我本git项目；



### 协议编码与解码

#### 走进源码看何处编码

在grpc-client端调用具体方法如 `search.pb.go#Search`时，最终是调用 `google.golang.org/grpc/call.go#Invoke`,对于我们的protobuf的message到这里会变成 `args, reply interface{}`，

真正执行代码为：

```go
//google.golang.org/grpc@v1.26.0/call.go:65#invoke
func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
	cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
	if err != nil {
		return err
	}
	if err := cs.SendMsg(req); err != nil {
		return err
	}
	return cs.RecvMsg(reply)
}
```

在 `google.golang.org/grpc@v1.26.0/stream.go:680#SendMsg`，在发消息给provider时，会先把结构体编码成字节流，`google.golang.org/grpc@v1.26.0/stream.go:1513#prepareMsg`  中的`encode(codec, m)`



```
func encode(c baseCodec, msg interface{}) ([]byte, error) {
	if msg == nil { // NOTE: typed nils will not be caught by this check
		return nil, nil
	}
	b, err := c.Marshal(msg) //此处为ProtoMarshaller
	if err != nil {
		return nil, status.Errorf(codes.Internal, "grpc: error while marshaling: %v", err.Error())
	}
	if uint(len(b)) > math.MaxUint32 {
		return nil, status.Errorf(codes.ResourceExhausted, "grpc: message too large (%d bytes)", len(b))
	}
	return b, nil
}
```

可能有人会问在哪设置的是ProtoMarshaller？

在前面的时候 `newClientStream(ctx, unaryStreamDesc, cc, method, opts...)`里面有个`setCallInfoCodec(c)`，里面会 `c.codec = encoding.GetCodec(proto.Name)`，在`proto.go`会注册进去，回归正题。



进入到`github.com/grpc-ecosystem/grpc-gateway@v1.12.1/runtime/marshal_proto.go:20`中的 `proto.Marshal(message)`方法，

```go
// Marshal takes a protocol buffer message
// and encodes it into the wire format, returning the data.
// This is the main entry point.
func Marshal(pb Message) ([]byte, error) {
	if m, ok := pb.(newMarshaler); ok {
		siz := m.XXX_Size()
		b := make([]byte, 0, siz)
		return m.XXX_Marshal(b, false)
	}
	if m, ok := pb.(Marshaler); ok {
		// If the message can marshal itself, let it do it, for compatibility.
		// NOTE: This is not efficient.
		return m.Marshal()
	}
	// in case somehow we didn't generate the wrapper
	if pb == nil {
		return nil, ErrNil
	}
	var info InternalMessageInfo
	siz := info.Size(pb)
	b := make([]byte, 0, siz)
	return info.Marshal(b, pb, false)
}
```

看下`info.Marshal(b, pb, false)`具体实现：

```go
// Marshal is the entry point from generated code,
// and should be ONLY called by generated code.
// It marshals msg to the end of b.
// a is a pointer to a place to store cached marshal info.
func (a *InternalMessageInfo) Marshal(b []byte, msg Message, deterministic bool) ([]byte, error) {
	u := getMessageMarshalInfo(msg, a)
	ptr := toPointer(&msg)
	if ptr.isNil() {
		// We get here if msg is a typed nil ((*SomeMessage)(nil)),
		// so it satisfies the interface, and msg == nil wouldn't
		// catch it. We don't want crash in this case.
		return b, ErrNil
	}
	return u.marshal(b, ptr, deterministic)
}
```

其实看到这个 `InternalMessageInfo`结构，其实在我们上面生成的`search.pb.go`就有过调用，如：

```go

func (m *SearchRequest) XXX_Unmarshal(b []byte) error {
	return xxx_messageInfo_SearchRequest.Unmarshal(m, b)
}
func (m *SearchRequest) XXX_Marshal(b []byte, deterministic bool) ([]byte, error) {
	return xxx_messageInfo_SearchRequest.Marshal(b, m, deterministic)
}
func (m *SearchRequest) XXX_Merge(src proto.Message) {
	xxx_messageInfo_SearchRequest.Merge(m, src)
}
func (m *SearchRequest) XXX_Size() int {
	return xxx_messageInfo_SearchRequest.Size(m)
}
func (m *SearchRequest) XXX_DiscardUnknown() {
	xxx_messageInfo_SearchRequest.DiscardUnknown(m)
}

var xxx_messageInfo_SearchRequest proto.InternalMessageInfo
```

也就是说，大多数后续操作都封装到了`table_marshal.go`中核心执行函数`(u *marshalInfo) marshal(b []byte, ptr pointer, deterministic bool)`

```go
// marshal is the main function to marshal a message. It takes a byte slice and appends
// the encoded data to the end of the slice, returns the slice and error (if any).
// ptr is the pointer to the message.
// If deterministic is true, map is marshaled in deterministic order.
func (u *marshalInfo) marshal(b []byte, ptr pointer, deterministic bool) ([]byte, error) {
	if atomic.LoadInt32(&u.initialized) == 0 {
		u.computeMarshalInfo()
	}

	// If the message can marshal itself, let it do it, for compatibility.
	// NOTE: This is not efficient.
	if u.hasmarshaler {
		m := ptr.asPointerTo(u.typ).Interface().(Marshaler)
		b1, err := m.Marshal()
		b = append(b, b1...)
		return b, err
	}

	var err, errLater error
	// The old marshaler encodes extensions at beginning.
	if u.extensions.IsValid() {
		e := ptr.offset(u.extensions).toExtensions()
		if u.messageset {
			b, err = u.appendMessageSet(b, e, deterministic)
		} else {
			b, err = u.appendExtensions(b, e, deterministic)
		}
		if err != nil {
			return b, err
		}
	}
	if u.v1extensions.IsValid() {
		m := *ptr.offset(u.v1extensions).toOldExtensions()
		b, err = u.appendV1Extensions(b, m, deterministic)
		if err != nil {
			return b, err
		}
	}
	for _, f := range u.fields {
		if f.required {
			if ptr.offset(f.field).getPointer().isNil() {
				// Required field is not set.
				// We record the error but keep going, to give a complete marshaling.
				if errLater == nil {
					errLater = &RequiredNotSetError{f.name}
				}
				continue
			}
		}
		if f.isPointer && ptr.offset(f.field).getPointer().isNil() {
			// nil pointer always marshals to nothing
			continue
		}
		b, err = f.marshaler(b, ptr.offset(f.field), f.wiretag, deterministic)
		if err != nil {
			if err1, ok := err.(*RequiredNotSetError); ok {
				// Required field in submessage is not set.
				// We record the error but keep going, to give a complete marshaling.
				if errLater == nil {
					errLater = &RequiredNotSetError{f.name + "." + err1.field}
				}
				continue
			}
			if err == errRepeatedHasNil {
				err = errors.New("proto: repeated field " + f.name + " has nil element")
			}
			if err == errInvalidUTF8 {
				if errLater == nil {
					fullName := revProtoTypes[reflect.PtrTo(u.typ)] + "." + f.name
					errLater = &invalidUTF8Error{fullName}
				}
				continue
			}
			return b, err
		}
	}
	if u.unrecognized.IsValid() {
		s := *ptr.offset(u.unrecognized).toBytes()
		b = append(b, s...)
	}
	return b, errLater
}
```

对于这段目前个人建议可以先放下，大致总结下，他是直接换取到属性的内存字节按照一定规则返回字节流；



#### 手动本地调试协议编码

接下来直接，输入message与我们返回的[]byte做个对比；在上面源码解析的时候，最终发现编解码调用其实就是在编译后的pb.go文件；

```go
import "github.com/golang/protobuf/proto"
func TestSearchRequest(t *testing.T) {
	ss := &SearchRequest{BizType: "123"}
	bs, _ := proto.Marshal(ss)
	fmt.Println(bs)
	// 第一个长度大小、第二个长度开始
	fmt.Println(string(bs))
}
```

最后我们发现它会循环遍历哪些被赋值的属性，

```go
for _, f := range u.fields {
		if f.required {
			if ptr.offset(f.field).getPointer().isNil() {
				// Required field is not set.
				// We record the error but keep going, to give a complete marshaling.
				if errLater == nil {
					errLater = &RequiredNotSetError{f.name}
				}
				continue
			}
		}
		if f.isPointer && ptr.offset(f.field).getPointer().isNil() {
			// nil pointer always marshals to nothing
			continue
		}
		b, err = f.marshaler(b, ptr.offset(f.field), f.wiretag, deterministic)
		if err != nil {
			if err1, ok := err.(*RequiredNotSetError); ok {
				// Required field in submessage is not set.
				// We record the error but keep going, to give a complete marshaling.
				if errLater == nil {
					errLater = &RequiredNotSetError{f.name + "." + err1.field}
				}
				continue
			}
			if err == errRepeatedHasNil {
				err = errors.New("proto: repeated field " + f.name + " has nil element")
			}
			if err == errInvalidUTF8 {
				if errLater == nil {
					fullName := revProtoTypes[reflect.PtrTo(u.typ)] + "." + f.name
					errLater = &invalidUTF8Error{fullName}
				}
				continue
			}
			return b, err
		}
	}
```

最终执行`TestSearchRequest`方法结果为：

```
[106 3 49 50 51]
```

如果是`BizType: "123"` 换成 其它字符串，最终发现 106 开头这个是不变的，后面跟着是它的长度，最后网上找了下资料，总结如下：



### **protobuf协议核心**

上面说了protobuf的message中有很多字段，其中字节流顺序是根据域号排序的，

每个字段的格式为：

> 修饰符 字段类型 字段名 = 域号;
> 在序列化时，protobuf按照TLV的格式序列化每一个字段，
>
> T即Tag，也叫Key；
>
> V是该字段对应的值value；L是Value的长度，
>
> 如果一个字段是整形，这个L部分会省略。
> 序列化后的Value是按原样保存到字符串或者文件中，Key按照一定的转换条件保存起来，序列化后的结果就是 KeyValueKeyValue…。Key的序列化格式是按照message中字段后面的域号与字段类型来转换。
>
> 转换公式如下：
>
> (field_number << 3) | wire_type

上面的field_number就是域号， wire_type与字段的类型有关

protobuf message是一系列的键值对，message的二进制形式使用字段的tag数值作为key，而其的字段名以及实际的数据类型则只能在解码端根据message的类型定义来决定。

编码message时，key和value被连接成字节流。当解码message的时候，解释器要有能力跳过那些他不能识别的字段，这样子，才能够添加新的字段进message而且不会影响不能识别这些新字段的老程序。

为此，每个字节流上的每个key实际上是由两个值组成的——一个是字段的tag number，另一个是
该字段的 `wire type`，这是用来提供足够的信息来明确该字段的值得长度的。



#### 可选的`wire type`：

| Type | Meaning          | Used For                                                 |
| ---- | ---------------- | -------------------------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups(已废弃)                                           |
| 4    | End group        | groups(已废弃)                                           |
| 5    | 32-bit           | fixed32, sfixed32, float                                 |

字节流上的每个key都是varint。key的值是（字段tag number<<3 | wire_type），即最后3个bit用来存储该字段的`wire type`。



#### Signed Integers

有符号整数类型(`sint32`、`sint64`)和“标准”整数类型(`int32`、`int64`)都是编码成varints，但是，当他们的值是一个负数时，会有很重要的不同。

当你使用`int32`和`int64`存储负数时，**总会产生一个长度为10个字节的varint**——它被看做是一个十分大的无符号整型。

而如果使用其中一种有符号整型(`sint32`、`sing64`)，则会使用ZigZag编码产生varint，效率比`int32`、`int64`高。

ZigZag 编码使用”zig-zags”在正负数间回退前进的方法，使-1编码成1，1编码成2，-2编码成3，如此类推：

| Signed Original | Encoded As |
| --------------- | ---------- |
| 0               | 0          |
| -1              | 1          |
| 1               | 2          |
| -2              | 3          |
| 2147483647      | 4294967294 |
| -2147483648     | 4294967295 |

换言之，每个值n都会根据以下公式进行编码：

对于sint32：
`(n << 1) ^ (n >> 31)`

对于sint64：
`(n << 1) ^ (n >> 63)`

注意，这里的第二个移位——`n >> 31` 部分，是算术移位，也就是说，如果n是正数，则移位的结果是所有位都是0，如果n是负数，则所有位都是1。

分析`sint32`或`sint64`时，其值会被解码成原始的有符号的形式。



#### protobuf协议解析

回归正题，上面我们有个message为SearchRequest对象，接下来我们通过赋值及对比代码输出的字节流：

主要研究 `SearchRequest` 生成的字节流

```protobuf
message SearchRequest {
    UrlVO url = 1;
    string bizType = 13;
    int32 runMode = 14;
}

message UrlVO {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
}
```

测试用例：

```go
func TestSearchRequest(t *testing.T) {
	url := &UrlVO{
		Url:   "https://github.com/Zeb-D",
		Title: "my-review",
	}
	ss := &SearchRequest{RunMode: 260, BizType: "123", Url: url}
	bs, _ := proto.Marshal(ss)
	fmt.Println(bs)
	// 第一个长度大小、第二个长度开始
	fmt.Println(string(bs))
	bs1,_:= proto.Marshal(url)
	fmt.Println(bs1)
}
```

为什么这样测试呢？

这里面内嵌了另外一个message即编译后的`UrlVO`，因为UrlVO对应的域号为1，按照规则先解析UrlVO：

我们先练习下解析

```go
url := &UrlVO{
		Url:   "https://github.com/Zeb-D",
		Title: "my-review",
	}
```

对于Url域号为1，类型为string，所以它的tag 为 `1<<3 | 2 = 10` ,length 为24，

对 `[]byte("https://github.com/Zeb-D")` 为 `104 116 116 112 115 58 47 47 103 105 116 104 117 98 46 99 111 109 47 90 101 98 45 68`

故Url 的TLV为`10 24 104 116 116 112 115 58 47 47 103 105 116 104 117 98 46 99 111 109 47 90 101 98 45 68`

对于Title域号为2，类型为string，所以它的tag 为 `2<<3 | 2 = 18` ,length 为9，value为`109 121 45 114 101 118 105 101 119` 故Title 的TLV为`109 121 45 114 101 118 105 101 119`；

总结UrlVO的对应的字节流为`10 24 104 116 116 112 115 58 47 47 103 105 116 104 117 98 46 99 111 109 47 90 101 98 45 68 18 9 109 121 45 114 101 118 105 101 119`



接下来解析最外层结构：

```go
ss := &SearchRequest{RunMode: 260, BizType: "123", Url: url}
```

这里260涉及到一个数值存在使用多个字节进行编码；

url域号为1，它的`type=2 embedded messages`，总结UrlVO的对应的字节流的length为37

故url的TLV为

```
10 37 10 24 104 116 116 112 115 58 47 47 103 105 116 104 117 98 46 99 111 109 47 90 101 98 45 68 18 9 109 121 45 114 101 118 105 101 119
```

bizType域号为13，type=2, tag为`13<<3 | 2 = 106`,length为3，value为`49 50 51`

故bizType的TLV为`106 3 49 50 51`

RunMode域号为14，type=0,tag为 `14<<3 | 2 = 112`，因为type是0，没有length这个字段浪费，

260 对应的bit位为

```
0000 0001 0000 0100
```

以8bit为单位，从后面转下顺序：

```
0000 0100 0000 0001
```

即这16bit是个整数字段，所以将前面的单位 第一个bit从0变1，因为开头有1表示后面的8bit是连续的

```
1000 0100 0000 0001
```

```
132 2
```

故RunMode的TV为

`112 132 2`



思考题比如整数32904这个应该怎么pb编码呢？

正常的bit表示：

```
1000 0000 1000 1000
```

以8bit为单位，从后面转下顺序：

```
1000 1000 1000 0000
```

因为这些前面都开头为1了，需要对应前面加bit为:

```
1000 1000  1000 0001 0000 0001
 000 1000   000 0001 0000 0001
0000 0001   000 0001 000 1000
        1   000 0001 000 1000 
1 000 1000  1   000 000
```

