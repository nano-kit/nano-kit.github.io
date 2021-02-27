快速开始
===

通过阅读本教程，你将学会

* 用 go-micro 开发一个微服务
* 用 micro api 网关导出服务 API
* 验证和调试服务 API

安装开发工具
---

我用的开发机是 MacOS v10.11.6 （OS X El Capitan），在这个计算机上，能安装的 Go 最高版本是 v1.14

> Go 1.14 is the last release that will run on macOS 10.11 El Capitan. Go 1.15 will require macOS 10.12 Sierra or later.

https://golang.org/doc/go1.14#darwin

Go 安装好之后，有关的环境变量如下

```sh
$ go env
GO111MODULE=""
GOARCH="amd64"
GOBIN=""
GOCACHE="/Users/huang/Library/Caches/go-build"
GOENV="/Users/huang/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOINSECURE=""
GONOPROXY=""
GONOSUMDB=""
GOOS="darwin"
GOPATH="/Users/huang/GoWork"
GOPRIVATE=""
GOPROXY="https://proxy.golang.org,direct"
GOROOT="/usr/local/Cellar/go@1.14/1.14.9/libexec"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/Cellar/go@1.14/1.14.9/libexec/pkg/tool/darwin_amd64"
GCCGO="gccgo"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/72/gdh8xjz13kz41rysd10rd9tw0000gn/T/go-build864495291=/tmp/go-build -gno-record-gcc-switches -fno-common"
```

除了 Go 之外，还需要安装

* 协议生成工具 `protoc-gen-micro` 和 `protoc-gen-go`
* 项目生成工具 `micro`

用 Protobuf 来定义服务 API 是开发过程中的最佳实践。协议生成工具能把 Protobuf 定义转换成 Go 代码，让项目直接使用。项目生成工具能生成新项目的骨架，不纠结目录安排和构建脚本，对新手非常友好。

首先下载 go-micro 和 micro 的代码到本地目录，

```
git clone https://github.com/nano-kit/go-micro.git
git clone https://github.com/nano-kit/micro.git
```

注意，这里没有去 https://github.com/micro/micro 下载，原因是 go-micro 从 v3.0 开始进行商业化的大改变，代码演进变化剧烈，不利于我们自己的项目使用。本教程用的是我们自己独立维护的 v2 稳定版，基于上游的 Go Micro v2.9.1

安装 `micro` 和 `protoc-gen-micro`

```
cd micro && go install
cd cmd/protoc-gen-micro && go install
```

安装 `protoc-gen-go`

```
cd $HOME; GO111MODULE=on go get github.com/golang/protobuf/protoc-gen-go@v1.3.2
```

命令执行成功后，它们将被安装在 $GOPATH/bin/ 下。

注意，安装 `protoc-gen-go` 的命令，必须先 cd 到 $HOME 下，否则 go get 会安装到当前所在项目的依赖里去。

> to avoid accidentally modifying go.mod, people started suggesting more complex commands

https://blog.golang.org/go116-module-changes

我们使用的是 protoc-gen-go@v1.3.2 ，新版的 `protoc-gen-go` 生成的 Go 代码要求使用最新的 google.golang.org/protobuf 而不是 github.com/golang/protobuf ，与 go-micro/v2 不能一起使用。

查看刚才安装好的开发工具

```sh
$ ls -lt $GOPATH/bin/
total 150264
-rwxr-xr-x  1 huang  staff  61932564 Feb 26 09:30 micro
-rwxr-xr-x  1 huang  staff   6277416 Feb 26 09:27 protoc-gen-go
-rwxr-xr-x  1 huang  staff   7789496 Feb 26 09:28 protoc-gen-micro
```

注意，$PATH 必须包括 $GOPATH/bin，也就是说下面这些命令都能在 $PATH 中找到

```sh
$ which go protoc protoc-gen-go protoc-gen-micro micro
/usr/local/opt/go@1.14/bin/go
/usr/local/bin/protoc
/Users/huang/GoWork/bin/protoc-gen-go
/Users/huang/GoWork/bin/protoc-gen-micro
/Users/huang/GoWork/bin/micro
```




生成骨架项目
---

新项目启动，可以用 `micro new` 生成骨架。需要指定的参数，有

* `--namespace` 服务的命名空间。建议一条业务线下的所有服务，用相同的命名空间，名称就用业务名
* `--alias` 服务的短名。建议遵循服务的功能，登录服务可以命名为 login，订单服务可以命名为 order
* `项目目录` 会自动创建指定的目录，以及里面的源代码文件

```sh
$ micro new --namespace com.example --alias realworld realworld-example-app
Creating service com.example.service.realworld in realworld-example-app

.
├── main.go
├── generate.go
├── plugin.go
├── handler
│   └── realworld.go
├── subscriber
│   └── realworld.go
├── proto
│   ├── realworld
│   │   └── realworld.proto
│   └── imports
│       └── api.proto
├── Dockerfile
├── Makefile
├── README.md
├── .gitignore
└── go.mod


compile the proto file realworld.proto:

	cd realworld-example-app
	make proto

build the project:

	make build
```

进入项目的目录，用 `make build` 构建，并运行 `./realworld-service`，微服务启动成功了。

上面的步骤，已经自动生成了这个微服务的接口协议，是用 Protobuf 语言来描述的，

```protobuf
syntax = "proto3";

package com.example.service.realworld;

service Realworld {
	rpc Call(Request) returns (Response) {}
    // 省略了流式调用的接口，与本教程无关
    // ...
}

message Request {
	string name = 1;
}

message Response {
	string msg = 1;
}

// Message 用于异步消息
message Message {
	string say = 1;
}
```

改动这个接口协议文件之后，可以用 `make proto` 生成新的 Go 代码。




把玩第一个微服务
---

保持 `./realworld-service` 在一个终端窗口中运行，我们打开另一个终端窗口来观察这个服务的特性。

### 查看服务注册表

```sh
$ micro list services
com.example.service.realworld
micro.http.broker
```

注册表里，有我们的服务 `com.example.service.realworld`。这是服务的 FQDN (Fully qualified domain name)，即网络上的唯一名称。它由 namespace.type.alias 组合而成。

注意，`micro.http.broker` 是由 go-micro/broker 悄悄注册的，用于消息投递。在[异步消息](asynchronous-messaging.md)这一章节里有详细解释。

### 查看服务详情

服务的详细信息有

* 版本号
* 实例ID
* 实例地址（Address）
* 元数据（Metadata）
* 接口（Endpoint）、入参（Request）、出参（Response）

```sh
$ micro get service com.example.service.realworld
service  com.example.service.realworld

version latest

ID	Address	Metadata
com.example.service.realworld-f653af63-1467-4379-a990-3dd51cad0e30	192.168.0.102:56203	protocol=grpc,registry=mdns,server=grpc,transport=grpc,broker=http

Endpoint: Realworld.Call

Request: {
	name string
}

Response: {
	msg string
}


Endpoint: Realworld.PingPong

Metadata: stream=true

Request: {}

Response: {}


Endpoint: Realworld.Stream

Metadata: stream=true

Request: {}

Response: {}


Endpoint: Realworld.Handle

Metadata: subscriber=true,topic=com.example.service.realworld

Request: {
	say string
}

Response: {}

```

### 查看服务状态

服务的运行状态有

* 实例ID（node）
* 地址（address:port）
* 启动时刻（started）
* 在线时长（uptime）
* 处理请求数量（requests）
* 占用内存（memory）
* 活跃线程（goroutine）数量（threads）
* 垃圾回收耗时（gc）
* 处理出错数量（errors）

```sh
$ micro stats com.example.service.realworld
service  com.example.service.realworld

version latest

node		address:port		started	uptime	requests	memory	threads	gc	errors
com.example.service.realworld-f653af63-1467-4379-a990-3dd51cad0e30		192.168.0.102:56203		Feb 27 09:47:18	44m37s	11	2.22mb	25	2.304788ms	0
```

### 调用服务的接口

`micro call` 的参数依次填写

* 服务在网络上的唯一名称 FQDN
* 接口（Endpoint）名称
* 入参（Request）的值

```sh
$ micro call com.example.service.realworld Realworld.Call '{"name":"Jack"}'
{
	"msg": "Hello Jack"
}
```

### 给服务发异步消息

`micro publish` 的参数依次填写

* 消息主题 topic
* 消息内容

```sh
$ micro publish com.example.service.realworld '{"say": "Hello!"}'
ok
```





用 micro api 网关导出服务 API
---

通常，客户端需要服务提供 REST API。按照[微服务的最佳实践](services-architecture.md)，客户端只用统一连接到服务网关（API Gateway），由服务网关自动发现后端微服务，转发请求。

基于 go-micro 开发的微服务，与 micro api 网关配合一起使用，是最方便的。启动网关的命名是，

```
micro --auth_namespace com.example api --namespace com.example --type service
```

启动的参数，指定了这个网关后端的服务都在 com.example.service 命名空间下。网关自己默认监听在 0.0.0.0:8080 这个地址。

用 curl 测试网关，

```
$ curl -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jack"}'
{"msg":"Hello Jack"}
```

请求路径的规则是，

* /realworld/Realworld/Call -> service=realworld, method=Realworld.Call

如果后端服务成功处理了请求，HTTP 的状态码总是 200 OK。如果后端服务在处理过程中出错，[错误处理的最佳实践](rpc-error-handling.md)是由处理方法用 "github.com/micro/go-micro/v2/errors" 这个包里定义的函数生成自定义错误，例如：

```go

    // Inside Realworld.Call handler

    // ...

    if req.Age == 0 {
		return errors.BadRequest("zero-age", "age is %v, forget to set the age?", req.Age)
	}

    if req.Age > 200 {
		return errors.New("age-too-old", fmt.Sprintf("age is %v, too old!", req.Age), 501)
	}

    // ...

```

```sh
$ curl -v -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jacket", "age":201}'
< HTTP/1.1 501 Not Implemented
{"id":"age-too-old","code":501,"detail":"age is 201, too old!","status":"Not Implemented"}

$ curl -v -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jacket"}'
< HTTP/1.1 400 Bad Request
{"id":"zero-age","code":400,"detail":"age is 0, forget to set the age?","status":"Bad Request"}
```

也就是说，客户端的处理逻辑是，

* 200 OK -> 按 Response 协议规格处理请求的响应
* 其它 -> 按 Error 协议规格处理出错
  - id 是错误的唯一ID
  - detail 是错误的描述

服务端需要注意，code 只能填写[已知的 HTTP status codes](https://golang.org/pkg/net/http/#pkg-constants)

这种约定的好处是，

* 服务的接口协议更简洁，Response 只需要描述业务需要的字段
* 错误的处理方式更自然，贴近 REST API 规范
