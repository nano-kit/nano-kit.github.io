流式 RPC
===

go-micro 支持流式 RPC。其协议的定义，与 gRPC 相同：

* 服务端流式 RPC 由客户端首先发送请求，并拿到一条流来依次读取多个应答。客户端不断的读取这条流，直到再也没有应答。

      rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);

* 客户端流式 RPC 由客户端依次在一条流上发送多个请求。发送完毕后，等待服务端处理完成发回应答。

      rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);

* 双向流式 RPC 是全双工的读写流。

      rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);

流式 RPC 的适用场景并不多见，但却不可缺少。

|                             |         适用场景         | 如果只能用 Unary RPC |
| --------------------------- | ------------------------ | -------------------- |
| Server streaming RPC        | 下载文件；获取巨量的数据 | 分页                 |
| Client streaming RPC        | 上传文件；传输巨量的数据 | 分批                 |
| Bidirectional streaming RPC | 实时订阅通知             | 轮询                 |


流式 RPC 的另一个好处，是让通信协议和处理逻辑更加简单自然、实时。由于一元 RPC 总是会有最大处理时间的限制（go-micro 的默认请求超时是 5 秒），它无法处理巨量的数据和耗时的处理过程。流式 RPC 通过渐进的数据处理，没有超时时间的限制。

流式 RPC 的缺陷是会独占一条 TCP 连接。而且，要处理断点续传。

go-micro 对流式 RPC 的支持非常强大。服务网关 go.micro.api 能把用 WebSocket 协议连入的客户端，与后端服务的流式接口协议无缝对接起来。这个例子在 [nano-kit/realworld-example-app/cmd/portal](https://github.com/nano-kit/realworld-example-app/tree/main/cmd/portal)
