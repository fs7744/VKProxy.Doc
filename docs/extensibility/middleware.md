# 如何通过中间件定制化功能

中间件是一种装配到应用管道以处理请求和响应的软件。 每个组件：

- 选择是否将请求传递到管道中的下一个组件。
- 可在管道中的下一个组件前后执行工作。

请求委托用于生成请求管道。 请求委托处理每个 HTTP/tcp/udp 请求。

具体概念可参考[ASP.NET Core 中间件](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/?view=aspnetcore-9.0)

这里列举一下涉及的三种中间件

## HTTP 中间件

HTTP 中间件其实就是[ASP.NET Core 中间件](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/?view=aspnetcore-9.0)

![](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/media/yarp-pipeline.png?view=aspnetcore-9.0)

只不过VKProxy 要兼容tcp 和udp 这样的L4代理，而ASP.NET Core只是针对http涉及，所以在入口配置处使用了差异的方式，不直接基于ASP.NET Core，避免影响L4代理

### HTTP 中间件定义

委托形式的
``` csharp
namespace Microsoft.AspNetCore.Http;

public delegate Task RequestDelegate(HttpContext context);

using Middleware = System.Func<Microsoft.AspNetCore.Http.RequestDelegate, Microsoft.AspNetCore.Http.RequestDelegate>;
```
或者 接口形式的
``` csharp
namespace Microsoft.AspNetCore.Http;

public interface IMiddleware
{
    Task InvokeAsync(HttpContext context, RequestDelegate next);
}
```



## UDP 中间件

## TCP 中间件

最后还有一个socks5的示例以供大家参考[如何利用中间件扩展实现socks5](/VKProxy.Doc/docs/extensibility/socks5)