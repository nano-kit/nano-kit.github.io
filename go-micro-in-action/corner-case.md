Corner Case 一些边角问题
===

这里记录在使用 go-micro 的过程中，遇到的一些边角问题。



JSON: int64 类型
---

> The Number.MAX_SAFE_INTEGER constant represents the maximum safe integer in JavaScript (2^53 - 1).
>
> -- [JavaScript reference - Number.MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER)

JavaScript 里最大的整数值为 9007199254740991，而 int64 最大的整数值为 9223372036854775807。当 Protobuf 定义里出现 int64 类型的字段时，从网页里的 js 发起的调用，会拿到字符串类型的响应。

举一个例子：

这是我们的 Protobuf 协议

```protobuf
message Response {
	string msg = 1;
	int32 num_int32 = 2;
	int64 num_int64 = 3;
	float num_float = 4;
	double num_double = 5;
}
```

在 handler 里，返回字段类型允许的最大数值，

```go
	rsp.NumInt32 = math.MaxInt32
	rsp.NumInt64 = math.MaxInt64
	rsp.NumFloat = math.MaxFloat32
	rsp.NumDouble = math.MaxFloat64
```

通过服务网关调用接口，输出响应

```sh
$ curl -s -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jack","age":2}' | jq
{
  "msg": "Hello Jack",
  "num_int32": 2147483647,
  "num_int64": "9223372036854775807",
  "num_float": 3.4028235e+38,
  "num_double": 1.7976931348623157e+308
}
```

所以，go-micro 的 JSON 编解码器帮我们避免了这个坑，防止了 int64 类型的数值在 JSON 转换时精度丢失。

这是怎么做到的呢？

> Package jsonpb provides functionality to marshal and unmarshal between a protocol buffer message and JSON. It follows the specification at https://developers.google.com/protocol-buffers/docs/proto3#json.
>
> -- [source code of package jsonpb](https://github.com/golang/protobuf/blob/v1.4.3/jsonpb/encode.go#L547)

也就是说，只要采用 package jsonpb ，而不是标准库里的
encoding/json 来做转换，就会遵循 Proto3 JSON Mapping 规范。这个规范规定

|         proto3         |  JSON  | JSON example |                                    Notes                                     |
| ---------------------- | ------ | ------------ | ---------------------------------------------------------------------------- |
| int64, fixed64, uint64 | string | "1", "-10"   | JSON value will be a decimal string. Either numbers or strings are accepted. |

处理这个规则的代码是

```go
    case int64, uint64:
		w.write(fmt.Sprintf(`"%d"`, v.Interface()))
```


JSON: omitempty
---

> The "omitempty" option specifies that the field should be omitted from the encoding if the field has an empty value, defined as false, 0, a nil pointer, a nil interface value, and any empty array, slice, map, or string.
>
> -- [encoding/json.Marshal](https://golang.org/pkg/encoding/json/#Marshal)

由 Protobuf 生成的 Go 协议文件，会对结构体的每个字段都自动加上 omitempty

```go
type Response struct {
	Msg                  string   `protobuf:"bytes,1,opt,name=msg,proto3" json:"msg,omitempty"`
	NumInt32             int32    `protobuf:"varint,2,opt,name=num_int32,json=numInt32,proto3" json:"num_int32,omitempty"`
	NumInt64             int64    `protobuf:"varint,3,opt,name=num_int64,json=numInt64,proto3" json:"num_int64,omitempty"`
	NumFloat             float32  `protobuf:"fixed32,4,opt,name=num_float,json=numFloat,proto3" json:"num_float,omitempty"`
	NumDouble            float64  `protobuf:"fixed64,5,opt,name=num_double,json=numDouble,proto3" json:"num_double,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```

如果这个结构体的某一个字段值恰好是 “empty value”，那么转换成 JSON 之后，这个字段不会出现在结果中。JavaScript 代码里访问这个字段会 `undefined`，甚至会报错 `TypeError: Cannot read property 'x' of undefined`。

想在 JavaScript 里优雅的解决这个问题非常麻烦。可是在服务端去修复，则非常简单。

```go
var jsonpbMarshaler = &jsonpb.Marshaler{
	EnumsAsInts:  false,
	EmitDefaults: true, // convenient for js
	OrigName:     true,
}
```

服务端生成的 JSON 结果由 jsonpbMarshaler 控制，一旦这个 `EmitDefaults` 选项设置成 `true`，所有的响应 JSON 都会加上 “empty value” 字段。客户端可以按照常规方式访问它们。

在 nano-kit/go-micro 仓库里已经做了这个修改。这样改的后果是，响应数据会增多，消耗更多的流量。应对的办法是，启用 HTTP Cache-Control 和 Compression。



Idempotent Interface 接口的幂等性
---

思考接口幂等性的意义是，

* 防止编程库自动重试，造成系统损失，例如重复支付
* 防止人为重放攻击，例如充值

实现幂等的常用思路，

### MVCC

多版本并发控制，乐观锁的一种实现，在数据更新时需要去比较持有数据的版本号，版本号不一致的操作无法成功。例如博客点赞次数自动+1的接口：

```go
func AddCount(id, version int64) error
```

```sql
update blog set count=count+1, version=version+1 where id=2021 and version=302
```

每一个version只有一次执行成功的机会，一旦失败必须重新获取。

### Unique Index 去重表

利用数据库表单的特性来实现幂等，常用的一个思路是在表上构建唯一性索引，保证某一类数据一旦执行完毕，后续同样的请求再也无法成功写入。

例子还是上述的博客点赞问题，要想防止一个人重复点赞，可以设计一张表，将博客id与用户id绑定建立唯一索引，每当用户点赞时就往表中写入一条数据，这样重复点赞的数据就无法写入。

### Token 机制

这种机制就比较重要了，适用范围较广，有多种不同的实现方式。其核心思想是为每一次操作生成一个唯一性的凭证，也就是 token。一个 token 在操作的每一个阶段只有一次执行权，一旦执行成功则保存执行结果。对重复的请求，返回同一个结果。

以电商平台为例子，电商平台上的订单 ID 就是最适合的 token。当用户下单时，会经历多个环节，比如生成订单，减库存，减优惠券等等。

每一个环节执行时都先检测一下该订单 ID 是否已经执行过这一步骤，对未执行的请求，执行操作并缓存结果，而对已经执行过的 ID，则直接返回之前的执行结果，不做任何操作。这样可以在最大程度上避免操作的重复执行问题，缓存起来的执行结果也能用于事务的控制等。
