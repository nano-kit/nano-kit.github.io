一些边角问题
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

这样做会导致一个问题：如果字段值恰好是 `empty value`，那么转换成 JSON 之后，这个字段不会出现在结果中。JavaScript 代码里访问这个字段会 `undefined`，甚至会报错 `TypeError: Cannot read property 'x' of undefined`。

想在 JavaScript 里优雅的解决这个问题非常麻烦。可是在服务端去修复，则非常简单。

```go
var jsonpbMarshaler = &jsonpb.Marshaler{
	EnumsAsInts:  false,
	EmitDefaults: true, // convenient for js
	OrigName:     true,
}
```

