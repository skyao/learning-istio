# Mixer Adapter Handler

Handler用来配置Mixer Adaper，或者直白说：Handler就是适配器的配置。

> Mixer 使用的每个适配器都需要一些配置来进行操作。一般来说适配器需要一些数据进行实例化，例如后端的 URL、认证信息、缓存选项等等。每个适配器使用 [protobuf](https://developers.google.com/protocol-buffers/) 消息来定义所需的配置数据。
> 
> 可以通过创建 [Handler](http://istio.doczh.cn/docs/concepts/policy-and-control/mixer-config.html#handlers) 的方式来为适配器提供配置。Handler 就是一套能让适配器就绪的完整配置。对同一个适配器可以有任意数量的 Handler，这样就可以在不同场景下复用。

## 定义

Handler的定义，这是一个接口：

```go
// Handler表示每个Adapter必须实现的默认功能
type Handler interface {
    io.Closer
}
```

只有简单的声明实现了 `io.Closer` 接口。

还定义了一个 HandlerBuilder 接口：

```go
// HandlerBuilder 代表handler的工厂。
// Adapter向Mixer注册builder以便允许Mixer根据需要实例化handler。
// 对于给定的builder，Mixer调用各种特定于模板的SetXXX方法，SetAdapterConfig方法，
// 一旦完成，Mixer将调用Validate，然后调用Build方法。
// Build方法返回一个handler，在请求处理期间Mixer会调用该handler
type HandlerBuilder interface {
    // SetAdapterConfig 给予builder adapter级别的配置状态
    SetAdapterConfig(Config)

    // Validate负责确认提供给builder的所有配置是正确的。
    // 仅当Validate方法返回成功时才调用Build方法。
    Validate() *ConfigErrors

    // Build必须返回Handler。
    // 这个handler要实现所有builder配置的特定于模板的运行时请求服务接口。
    // 这意味着Build方法返回的handler必须为Adapter支持的所有模板实现所有运行时接口。
    // 如果返回的 Handler 没能实现build注册要求的接口，
    // Mixer将报告错误并停止服务运行时流量到特定handler。
    Build(context.Context, Env) (Handler, error)
}
```

