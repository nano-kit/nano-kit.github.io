远程调用（RPC）的错误处理
===

我们写下的每一行代码，都可能会“出错”。这里的“出错”，指的是代码的功能无法完成，结果不符合预期，当然也包含了人为犯错的意思。

错误处理，就是检查这些错误，及时通知执行者，进而采取相应的处理手段，是忽略、跳转、重试还是终止。

有些编程语言，采用异常机制来实行错误处理，如 Java, Python。另一些编程语言，采用显式的 err 返回值来实行错误处理，如 Go。人们对异常机制多有诟病，说它“有隐含的控制流程，让程序员很难追溯”。显式错误处理，对编写可靠软件会更有帮助。

远程调用（RPC）的每个接口，一样的，也必须有错误处理。不同于函数调用，每个远程调用都会因为网络不通导致失败。一种画蛇添足式的 RPC 错误处理，是给每个接口的返回值添加错误码：

```protobuf
message Response {
    int32  error_code    = 1;
    string error_message = 2;
    // 有用的数据，从这里开始
    string application_reply = 3;
}
```

它无形中加重了使用者的认知负担：

* 由于每个接口都可能失败，实现接口的一方，不但要记得在每个返回值结构体添加错误码字段，
* 还要在具体实现中，把编程语言的内建错误处理机制（异常或者err）转换成 error_code 和 error_message，
* 调用接口的一方，不但要关注远程链路上可能的错误（网络中断、处理超时等等无法收到 Response 从而 error_code 不可用），
* 还要关注每个返回值的 error_code，用不到编程语言的内建错误处理机制。

这些缺点，让 RPC 的错误处理非常繁琐。即使再发明代码生成工具，自动检查工具，封装函数库，也是在错误的道路上，一错再错，不符合“如无必要，勿增实体”的设计理念。

那么，RPC 的错误处理，到底应该怎么做？

gRPC 早就给出了标准答案：

* 要能利用编程语言的内建错误处理机制，不给使用者额外的负担。远程调用，应该被看作可能会出错的本地调用，不应该被区分对待。
* Protobuf 协议只应该描述应用数据。错误处理，是远程调用编程框架与使用者的统一约定。

佐证的例子有：

* [Public interface definitions of Google APIs](https://github.com/googleapis/googleapis)
* [A basic tutorial introduction to gRPC in Go](https://www.grpc.io/docs/languages/go/basics/)

Go-Micro 的错误处理
---

Go-Micro 的错误处理符合直觉。如果服务实现方，在 handler 里返回 Go 原生 error，如

```go
return errors.New("bad request")
return fmt.Errorf("invalid parameter: %v", param)
```

Go 客户端能拿到 `err.Error()` 是 JSON 的 error，同样的 REST 客户端能拿到这个 JSON，并且 HTTP 错误码与 `code` 一致。

Go-Micro 与使用者约定的错误规范是

```protobuf
// https://github.com/nano-kit/go-micro/tree/main/errors

message Error {
  string id = 1;     // 唯一ID，业务可以用 <MODULE>-1023 这种带前缀的编号，Go-Micro 内部用服务名作ID
  int32 code = 2;    // 与 HTTP 错误码一致 https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
  string detail = 3; // 详细的错误信息
  string status = 4; // 一般是 http.StatusText(code) https://golang.org/pkg/net/http/#StatusText
}
```

服务端也能利用 `"github.com/micro/go-micro/v2/errors"` 这个包构造符合规范的错误，例如

```go
return errors.BadRequest("E2003 zero-age", "age is %v, forget to set the age?", req.Age)
return errors.New("E2004 age-too-old", fmt.Sprintf("age is %v, too old!", req.Age), 520)
```

客户端如果有需要，能用 `errors.FromError` 解析出 Go-Micro 的 Error 结构。

扩展阅读资料
---

* [别只检查错误，要正确处理它们](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)
* [错误标记：替代“面向行为进行错误断言”的方案](https://npf.io/2021/04/errorflags/)
