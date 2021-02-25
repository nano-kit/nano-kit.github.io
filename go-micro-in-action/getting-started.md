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

除了 Go 之外，还需要安装协议生成工具 `protoc-gen-micro` 和 `protoc-gen-go`。

用 Protobuf 来定义服务 API 是开发过程中的最佳实践。协议生成工具能把 Protobuf 定义转换成 Go 代码，让项目直接使用。

首先下载 go-micro 和 micro 的代码到本地目录，

```
git clone https://github.com/nano-kit/go-micro.git
git clone https://github.com/nano-kit/micro.git
```

注意，这里没有去 https://github.com/micro/micro 下载，原因是 go-micro 从 v3.0 开始进行商业化的大改变，代码演进变化剧烈，不利于我们自己的项目使用。本教程用的是我们自己独立维护的 v2 稳定版，基于上游的 Go Micro v2.9.1

安装 `protoc-gen-micro`

```
cd micro/cmd/protoc-gen-micro && go install
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
total 332024
-rwxr-xr-x  1 huang  staff   6281816  2 25 23:31 protoc-gen-go
-rwxr-xr-x  1 huang  staff   7789496  2 25 23:17 protoc-gen-micro
```

注意，$PATH 必须包括 $GOPATH/bin，也就是说下面这些命令都能在 $PATH 中找到

```sh
$ which go protoc protoc-gen-go protoc-gen-micro
/usr/local/opt/go@1.14/bin/go
/usr/local/bin/protoc
/Users/huang/GoWork/bin/protoc-gen-go
/Users/huang/GoWork/bin/protoc-gen-micro
```




编写服务接口
---

前面说过，定义服务 API 的最佳实践是用 Protobuf 语言。这是服务接口代码，

```protobuf
syntax = "proto3";

```
