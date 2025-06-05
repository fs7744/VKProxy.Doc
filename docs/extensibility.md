# 定制化扩展

为了方便大家使用 KVProxy 在一些场景，默认功能无法满足时，可以通过自定义扩展实现自己的需求。

同时也是遵照 asp.net core 设计理念，提供了两种扩展方式

## 中间件管道

中间件是一种装配到应用管道以处理请求和响应的软件。 每个组件：

- 选择是否将请求传递到管道中的下一个组件。
- 可在管道中的下一个组件前后执行工作。

请求委托用于生成请求管道。 请求委托处理每个 HTTP/tcp/udp 请求。

具体可参考[ASP.NET Core 中间件](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/?view=aspnetcore-9.0)

KVProxy 添加了 udp 和 tcp 的特殊中间件

具体参见[如何通过中间件定制化功能](/VKProxy.Doc/docs/extensibility/middleware)

还有一个socks5的示例以供大家参考[如何利用中间件扩展实现socks5](/VKProxy.Doc/docs/extensibility/socks5)

## 特定功能策略增加

有些特定功能策略比较难以直接使用中间件扩展，这里列举主要部分 

（其实由于基于依赖注入，天生解耦，所以内部实现基本都可以覆盖或者添加新实现）

- [如何扩展服务发现](/VKProxy.Doc/docs/extensibility/destinationresolver)
- [如何扩展负载均衡策略](/VKProxy.Doc/docs/extensibility/loadbalancingpolicy)
- [如何扩展主动健康检查策略](/VKProxy.Doc/docs/extensibility/activehealthchecker)
- [如何扩展HTTP转换器](/VKProxy.Doc/docs/extensibility/transform)