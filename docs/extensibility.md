# 定制化扩展

为了方便大家使用 KVProxy 在一些场景，默认功能无法满足时，可以通过自定义扩展实现自己的需求。

同时也是遵照 asp.net core 设计理念，提供了两种扩展方式

## 中间件管道

中间件是一种装配到应用管道以处理请求和响应的软件。 每个组件：

- 选择是否将请求传递到管道中的下一个组件。
- 可在管道中的下一个组件前后执行工作。

请求委托用于生成请求管道。 请求委托处理每个 HTTP/tcp/udp 请求。

具体可参考[ASP.NET Core 中间件](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/?view=aspnetcore-9.0)

## 特定功能策略增加