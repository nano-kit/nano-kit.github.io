服务版本
===

 服务的版本化，有两种用法。

 * Registry 服务注册时，指定版本
 * RPC Resolver 服务注册时，用带有版本的服务名

Registry
---

go-micro 服务启动时，会向注册中心登记自己的信息

* Name
* Version
* Node
* Endpoints

micro get service 向注册中心获取信息，用树形组织

* Name
  * Version A
    * Nodes
  * Version B
    * Nodes

go-micro client 调用时，用 FilterVersion 指定目标


RPC Resolver
---

服务网关按如下规则路由请求到服务端点

假设后端服务名 (com.example.service.realworld) 和接口名 (Greeter.Hello)

URLs are resolved as follows:

|       Path       | Service | Method  |
| ---------------- | ------- | ------- |
| /foo/bar         | foo     | Foo.Bar |
| /foo/bar/baz     | foo     | Bar.Baz |
| /foo/bar/baz/cat | foo.bar | Baz.Cat |

Versioned API URLs can easily be mapped to service names:

|      Path       | Service | Method  |
| --------------- | ------- | ------- |
| /foo/bar        | foo     | Foo.Bar |
| /v1/foo/bar     | v1.foo  | Foo.Bar |
| /v1/foo/bar/baz | v1.foo  | Bar.Baz |
| /v2/foo/bar     | v2.foo  | Foo.Bar |
| /v2/foo/bar/baz | v2.foo  | Bar.Baz |
