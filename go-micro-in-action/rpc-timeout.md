远程调用（RPC）的超时处理
===

> At Google, we developed a context package that makes it easy to pass request-scoped values, cancelation signals, and deadlines across API boundaries to all the goroutines involved in handling a request.
>
> -- [Go Concurrency Patterns: Context](https://blog.golang.org/context)

控制 RPC 超时的标准办法是用 Go 标准库里的 `context`。

在一次交易过程中，通常需要有一个时间上限。比如，向账号里充值时，最多等待 1 分钟。如果 1 分钟之后，交易仍然没有完成，则取消交易。如果充值流程，涉及到多个子系统的 RPC 交互，就需要有一种机制，能

1. 指定最长处理时间；这也是用户需要等待的最长时间
1. 一旦超过时限，取消每个子系统里正在运行的流程

（1）能保证请求方不会停止响应，（2）能减轻系统负担，因为它们的“工作成果”不再被需要了。

一个简陋的 RPC 框架，常常没有实现机制（2），后果就是处理线程数激增，内存占用飙涨，甚至导致服务不可用，更严重的会导致雪崩效应，整个服务对外表现为不可用。

业务应用与数据库的交互，也是同样的道理。超时的时候，数据库里正在执行的任务，也应该被取消。但可惜的是，Go MySQL driver 还[不支持](https://github.com/go-sql-driver/mysql/issues/731)。

Set 设置 go-micro 的超时
---

go-micro 框架的默认超时是 5 秒，在代码 `client/client.go` 里指定

```go
    // DefaultRequestTimeout is the default request timeout
    DefaultRequestTimeout = time.Second * 5
```

使用时，不需要去设置这个值，应该用 context 的标准用法。

```go
    func slowOperationWithTimeout(ctx context.Context) (Result, error) {
        ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
        defer cancel()  // releases resources if slowOperation completes before timeout elapses
        return slowOperation(ctx)
    }

    func slowOperation(ctx context.Context) (Result, error) {
        var rsp Result
        err := client.Call(ctx, req, &rsp)
        return rsp, err
    }
```

context 里的实际生效的超时，是设置过的最早时间。

Why 为什么超时能传递给下游
---

go-micro client 在发起请求的时候，会根据 context 上下文里的超时，去设置 CallOptions.RequestTimeout

```go
    // check if we already have a deadline
    d, ok := ctx.Deadline()
    if !ok {
        // no deadline so we create a new one
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, callOpts.RequestTimeout)
        defer cancel()
    } else {
        // got a deadline so no need to setup context
        // but we need to set the timeout we pass along
        opt := client.WithRequestTimeout(time.Until(d))
        opt(&callOpts)
    }
```

上面这段代码在 `client/grpc/grpc.go` 里，意思是优先以 context.Deadline() 为准，并且会重置 RequestTimeout 让它与 Deadline 一致。如果 context 里没有 Deadline，才会用 RequestTimeout。所以从此之后， RequestTimeout 和 Deadline 两者就一致了。

为什么一定要 RequestTimeout 和 Deadline 两者一致呢？我们知道 Deadline 只能用来控制请求方的超时，它无法控制下游服务，以及下下游服务的处理时间。RequestTimeout 被用来告知下游，最长的处理时间是多少，下游据此构造出新的 context Deadline 再请求下下游……，于是整个调用链条涉及到的服务，都能遵循一致的超时时间了。这就是 Google 设计出 context 的初衷和最大的威力。

RequestTimeout 由 RPC 协议的 header 在网络上传递。

在请求方 `client/grpc/grpc.go`

```go
// set timeout in nanoseconds
header["timeout"] = fmt.Sprintf("%d", opts.RequestTimeout)
```

在接收方 `server/grpc/grpc.go`

```go
// timeout for server deadline
to := md["timeout"]
// set the timeout if we have it
if len(to) > 0 {
    if n, err := strconv.ParseUint(to, 10, 64); err == nil {
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, time.Duration(n))
        defer cancel()
    }
}
```

这就是 go-micro 超时的奥秘。
