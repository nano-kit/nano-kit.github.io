远程调用（RPC）的认证授权
===

通过阅读本教程，你将学会

* 了解什么是微服务的认证授权
* 搭建具备认证机制的 go-micro 服务系统
* 设置账号和权限规则
* 验证和调试具有认证机制的服务 API

本章节的实战操作，必须用 nano-kit 的 go-micro/v2 稳定版。如何安装可以参考[快速开始](getting-started.md)教程。

What 简介
---

一个实用的微服务框架，需要有内建的认证授权机制。具体来说，就是

* 认证，鉴别请求方的身份
* 授权，根据请求方的身份，以及事先定好的规则，批准请求方访问资源

每个 go-micro 服务，有三种认证的方式备选，由 `--auth` 启动参数指定

* `noop` 不写 `--auth` 时缺省可用
* `jwt` 需要配置公私钥，使用 [JSON Web Token](https://jwt.io/) 算法
* `service` 只需要配置公钥，使用认证服务 go.micro.auth

每种认证的方式，都有如下功能

* `Generate` 根据用户 ID、密码等，生成一个新账号
* `Grant` 新增一条访问规则
* `Revoke` 删除一条访问规则
* `Rules` 获取所有访问规则
* `Verify` 根据请求方身份和请求资源，进行权限检查
* `Inspect` 从访问令牌 access_token 还原出请求方身份
* `Token` 根据账号、密码或者 refresh_token 生成访问令牌 access_token

所有功能一起协作，就是 go-micro 框架的认证授权机制。它基于 [Role-based access control](https://en.wikipedia.org/wiki/Role-based_access_control)，核心算法在 `Verify` 里。值得注意的是，认证方式 `jwt` 的规则是临时的，需要在服务启动时写代码来装配，参见 `micro/cmd/cmd.go`。认证方式 `service` 的规则是持久化的，储存在 Store 里的，可以用 micro cli 或者写代码来实时修改。

```go
// micro/cmd/cmd.go

// add the system rules if we're using the JWT implementation
// which doesn't have access to the rules in the auth service
if (*cmd.DefaultCmd.Options().Auth).String() == "jwt" {
    for _, rule := range inauth.SystemRules {
        if err := (*cmd.DefaultCmd.Options().Auth).Grant(rule); err != nil {
            return err
        }
    }
}
```

上面的简介，看了之后可能会有点懵。没关系，下面我们搭建一个具有认证授权功能的 go-micro 系统，在实战中一步一步去理解。


Start 启动认证服务 go.micro.auth
---

环境变量必须有

* MICRO_AUTH_PRIVATE_KEY
* MICRO_AUTH_PUBLIC_KEY

启动命令

```sh
$ micro --auth jwt auth
```

这是我们的认证服务。这个服务作为账号、规则的存储和管理方，用 `jwt` 方式启动。它启动后，会以 `go.micro.auth` 为服务名，注册到服务发现中心。

公私钥可以用这个[项目](https://github.com/aclisp/godashboard/tree/master/insecure)里的命令来生成，然后用这个脚本把它们设置成环境变量。注意 macOS 用 `base64 -b0` Linux 用 `base64 -w0`

```sh
export MICRO_AUTH_PRIVATE_KEY=$(cat key.pem | base64 -b0)
export MICRO_AUTH_PUBLIC_KEY=$(cat cert.pem | base64 -b0)
```

服务启动时，框架的初始化代码会生成自己的服务账号 service account。服务账号是服务的身份，可以用来作服务到服务请求的认证授权。区别于用户请求的认证授权，服务到服务请求的认证授权不受用户身份的影响，只要是从某个服务发出的请求，都会通过授权，适用于粗粒度的授权规则。我们现在暂时还用不到它。

```go
// go-micro/util/auth/auth.go

// Generate generates a service account for and continually
// refreshes the access token.
func Generate(id string, name string, a auth.Auth) error {
    // ...
}
```

注意，缺省情况下认证服务用 memory store 来存储需要持久化的账号、规则等信息。生产环境下，可以配置为 file store 或者 sql database store。





Start 启动 API 网关
---

环境变量只需有

* MICRO_AUTH_PUBLIC_KEY

启动命令

```sh
$ micro --auth service api --namespace com.example --type service
```

这是 API 网关。这个服务作为用户请求的统一入口，会首先鉴别请求方的身份，授权，然后才是其它处理。用 `service` 方式启动，说明每次有请求到来，它都会去联系认证服务。如果设置了公钥环境变量，则 `Inspect`（从访问令牌 access_token 还原出请求方身份）可以本地直接验证，减少一次远程调用。

```go
// go-micro/auth/service/service.go

// Inspect a token
func (s *svc) Inspect(token string) (*auth.Account, error) {
    // try to decode JWT locally and fall back to srv if an error occurs
    if len(strings.Split(token, ".")) == 3 && s.jwt != nil {
        return s.jwt.Inspect(token)
    }

    // the token is not a JWT or we do not have the keys to decode it,
    // fall back to the auth service
    rsp, err := s.auth.Inspect(context.TODO(), &pb.InspectRequest{Token: token})
    if err != nil {
        return nil, err
    }
    return serializeAccount(rsp.Account), nil
}

```

另外，`Verify`（根据请求方身份和请求资源，进行权限检查）会缓存 30 秒规则，而 RBAC (Role-based access control) 这个算法实际上是在本地计算的。也能一定程度上减少向认证服务的远程调用。

```go
// go-micro/auth/rules/rules.go

// Verify an account has access to a resource using the rules provided. If the account does not have
// access an error will be returned. If there are no rules provided which match the resource, an error
// will be returned
func Verify(rules []*auth.Rule, acc *auth.Account, res *auth.Resource) error {
    // ...
}

```

Backend 后端服务
---

API 网关的后端服务，可以照常启动。也就是说，它们用的是 `noop` 方式。互相之间调用不会进行权限检查。由于有统一的请求入口（即 API 网关），位于后端的服务，通常不会开启认证机制，以保证调用性能。值得注意的是，来自网关的用户请求，在认证通过后，会在上下文里保存请求方的身份信息。

获取上下文里的请求方的身份信息，用这个函数

```go
// AccountFromContext gets the account from the context, which
// is set by the auth wrapper at the start of a call. If the account
// is not set, a nil account will be returned. The error is only returned
// when there was a problem retrieving an account
func AccountFromContext(ctx context.Context) (*Account, bool)
```

框架保证请求方的身份信息能在所有调用链条上一直传递下去。在业务逻辑里，可以根据它识别用户身份，作相关处理。这部分的原理在，比较复杂，我们暂时无需去理解，只要知悉并能运用就好。

```go
// go-micro/util/wrapper/wrapper.go

func (a *authWrapper) Call(ctx context.Context, req client.Request, rsp interface{}, opts ...client.CallOption) error

func AuthHandler(fn func() auth.Auth) server.HandlerWrapper
```

Play 发起 API 请求
---

现在模拟用户，用 curl 向 API 网关发起请求，能正常响应。

```sh
$ curl -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jack","age":1}'
```

用下面的命令，禁止匿名用户访问，`--namespace` 表示需要切换到 com.example 这个名字空间，缺省用来 login 的账号 ID 是 default，密码是 password。

```
$ micro login --namespace com.example default password
You have been logged in
```

login 成功之后，可以查看当前名字空间下的账号和权限规则。

```sh
$ micro auth list accounts
ID		Scopes		Metadata
default		admin		n/a

$ micro auth list rules
ID		Scope			Access		Resource		Priority
default		<public>		GRANTED		*:*:*			0
```

缺省的规则，允许匿名请求。为了关闭缺省的规则，我们增加一条规则，

```sh
$ micro auth create rule --scope '' --priority 1 --resource '*:*:*' --access denied deny-public
Rule created

$ micro auth list rules
ID			Scope			Access		Resource		Priority
deny-public		<public>		DENIED		*:*:*			1
default			<public>		GRANTED		*:*:*			0
```

再次用 curl 发出请求，被拒绝。说明刚才增加的规则生效了。

```sh
$ curl -i -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jack","age":1}'
HTTP/1.1 401 Unauthorized

Unauthorized request
```

接下来，我们增加一条规则，允许具名用户请求，访问范围是 normal。访问范围（Scope）类似用户组，一个用户可以有多个访问范围（Scope）。用参数 `--priority` 指定更大的优先级。

```sh
$ micro auth create rule --scope normal --priority 1000 --resource '*:*:*' normal-any
Rule created

$ micro auth list rules
ID			Scope			Access		Resource		Priority
normal-any		normal			GRANTED		*:*:*			1000
deny-public		<public>		DENIED		*:*:*			1
default			<public>		GRANTED		*:*:*			0
```

然后，创建一个新用户，ID 是 user001，设定其访问范围是 normal。

```sh
$ micro auth create account --secret 123456 --scopes normal user001
Account created: {"id":"user001","type":"","issuer":"com.example","metadata":null,"scopes":["normal"],"secret":"123456"}
```

创建成功后，获取用户 user001 的访问令牌。

```sh
$ micro token --secret 123456 user001
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoidXNlciIsInNjb3BlcyI6WyJub3JtYWwiXSwibWV0YWRhdGEiOm51bGwsImV4cCI6MTYxNTExNjk3NSwiaXNzIjoiY29tLmV4YW1wbGUiLCJzdWIiOiJ1c2VyMDAxIn0.mKYr04OltFaES3w5TMcpkgR6jTT-Y35Hd1x3wtexsA-3Go77JAOwwsI_FDrL2NAmhBxjxS_WCqtNm-oDNqM7LZNil9IJzJnNHsXJwZLfWewZaQDeqEMFLhTHVTJkV3R1gfnMAM7plYNff0Cmf32sVTUXg9LPTsVInGYLBj0CVZr8G2SFuyNvct5R7P5WbliaeaOO6OOmJND8l_AHA5MD_-8wEoG4wOv1vmXyS1wchHsJW4PAT0bKCzn9FmT6D2RNaCkqwgz1svGOVMZiFiKoiJIqZLINArQ7Xnw-Kdpen_TeAUBZHEE72XBpXh6jQy8QcPl8AoqLtQpighmnQBGA1g",
  "refresh_token": "27d75932-03e0-455e-8cf8-1ff31aa458d8",
}
```

用这个用户的访问令牌发起请求，也就是把访问令牌填充到 HTTP `Authorization` 头，能得到响应。

```sh
$ curl -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoidXNlciIsInNjb3BlcyI6WyJub3JtYWwiXSwibWV0YWRhdGEiOm51bGwsImV4cCI6MTYxNTExNjk3NSwiaXNzIjoiY29tLmV4YW1wbGUiLCJzdWIiOiJ1c2VyMDAxIn0.mKYr04OltFaES3w5TMcpkgR6jTT-Y35Hd1x3wtexsA-3Go77JAOwwsI_FDrL2NAmhBxjxS_WCqtNm-oDNqM7LZNil9IJzJnNHsXJwZLfWewZaQDeqEMFLhTHVTJkV3R1gfnMAM7plYNff0Cmf32sVTUXg9LPTsVInGYLBj0CVZr8G2SFuyNvct5R7P5WbliaeaOO6OOmJND8l_AHA5MD_-8wEoG4wOv1vmXyS1wchHsJW4PAT0bKCzn9FmT6D2RNaCkqwgz1svGOVMZiFiKoiJIqZLINArQ7Xnw-Kdpen_TeAUBZHEE72XBpXh6jQy8QcPl8AoqLtQpighmnQBGA1g' -XPOST http://127.0.0.1:8080/realworld/Realworld/Call -d '{"name":"Jack","age":1}'
```

如果请求被拒绝，可能是访问令牌过期了。用 `micro token` 命令再次获取新的访问令牌就可以了。

仔细观察 API 网关的调试日志，会发现触发验证通过的规则，以及请求用户。打开调试日志的方法是，在应用启动时，设置环境变量 MICRO_LOG_LEVEL=debug

```
2021-03-07 19:57:25  file=rules/rules.go:109 level=debug service=api verify ok: rule=&{ID:normal-any Scope:normal Resource:0xc0004ef500 Access:0 Priority:1000}, resource=&{Name:* Type:* Endpoint:*}, account=&{ID:user001 Type:user Issuer:com.example Metadata:map[] Scopes:[normal] Secret:}
```

在业务逻辑里，用 `AccountFromContext` 获取上下文里的请求方的身份信息，发现是 user001。至此，我们没有写一行代码，体验了 go-micro 框架的认证授权机制。而且，只有授权的用户，才能访问服务 API。


Rules 权限检查的算法
---

算法的代码在 `go-micro/auth/rules/rules.go`

输入

* 规则列表
* 请求用户
* 请求资源

输出

* 是否放行

过程

1. 依次检查规则列表里每条规则的 Type, Name, Endpoint 只保留与请求资源相关的，
1. 保留规则按优先级从大到小排序，
1. 对访问范围是任何人（包括匿名用户）的规则，检查是否放行，
1. 对访问范围是任何具名用户的规则，检查是否放行，
1. 对访问范围在请求用户 Scopes 里的规则，检查是否放行，
1. 直到没有任何规则适用，则拒绝。

OAuth 与第三方账号集成
---

项目 `realworld-example-app` [这里](https://github.com/nano-kit/realworld-example-app/blob/80aea6dd6d1d8eb48d6774eed93f2c081a465a3a/cmd/websocket-server/main.go#L180)有一个例子，将 go-micro auth 与 GitHub OAuth 集成起来，关键步骤有

1. 按 GitHub OAuth [文档](https://docs.github.com/en/developers/apps/authorizing-oauth-apps)接入
1. 根据 GitHub User Profile 生成 micro auth 系统内的账号
1. 获取账号的 Refresh Token，保存在客户端
1. 客户端用 Refresh Token 获取 Access Token，调用 API






Appendix
---

```
$ micro --store sqlite --auth jwt auth
$ micro login --namespace go.micro default password
$ micro auth create account --secret 123456 --scopes service api-gate
$ MICRO_LOG_LEVEL=debug micro --auth service --auth_id api-gate --auth_secret 123456 api --namespace com.example --type service
$ micro login --namespace com.example default password
$ micro auth list rules
ID				Scope			Access		Resource								Priority
realworld-stream		<public>		GRANTED		service:com.example.service.realworld:Realworld.Stream			100
realworld-pingpong		<public>		GRANTED		service:com.example.service.realworld:Realworld.PingPong		100
clubhouse-subscribe		<public>		GRANTED		service:com.example.service.realworld:Clubhouse.Subscribe		100
any-account			*			GRANTED		*:*:*									2
deny-public			<public>		DENIED		*:*:*									1
```
