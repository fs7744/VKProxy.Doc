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
// or
using Middleware = System.Func<Microsoft.AspNetCore.Http.HttpContext, Microsoft.AspNetCore.Http.RequestDelegate, Task>;
```
或者 接口形式的
``` csharp
namespace Microsoft.AspNetCore.Http;

public interface IMiddleware
{
    Task InvokeAsync(HttpContext context, RequestDelegate next);
}
```

### 依赖注入接口

可以通过以下方法直接注入


``` csharp
public static IServiceCollection UseHttpMiddleware(this IServiceCollection services, Func<RequestDelegate, RequestDelegate> middleware);

public static IServiceCollection UseHttpMiddleware(this IServiceCollection services, Func<HttpContext, RequestDelegate, Task> middleware);

public static IServiceCollection UseHttpMiddleware<T>(this IServiceCollection services, params object?[] args) where T : class;
```

### 示例

比如我们想通过log 记录请求开始结束

中间件实现：

``` csharp
public class EchoHttpMiddleware : IMiddleware
{
    private readonly ILogger<EchoHttpMiddleware> logger;

    public EchoHttpMiddleware(ILogger<EchoHttpMiddleware> logger)
    {
        this.logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var req = context.Request;
        logger.LogInformation($"begin {req.Protocol} {req.Host} {req.Path} {DateTime.Now}");
        await next(context);
        logger.LogInformation($"end {req.Protocol} {req.Host} {req.Path} {DateTime.Now}");
    }
}
```

注入中间件

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.UseHttpMiddleware<EchoHttpMiddleware>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

效果：

为了展示，开启了info级别的log，产线大家还是要考虑以下log level，不然可能log爆炸

``` csharp
info: EchoHttpMiddleware[0]
      begin HTTP/1.1 www.google.com /ws/d 2025/6/5 10:32:23
info: VKProxy.Server.ReverseProxy[14]
      Proxying to http://XXXX/ws/d HTTP/2 RequestVersionOrLower
info: VKProxy.Server.ReverseProxy[15]
      Received HTTP/1.1 response NotFound.
info: EchoHttpMiddleware[0]
      end HTTP/1.1 www.google.com /ws/d 2025/6/5 10:32:24
```

## UDP 中间件

### UDP 中间件定义

由于 L4 属于比较底层的协议，本身没有类似HTTP request / response 一来必须一回的明确区分，所以不方便提供HTTP中间件那样简单的管道形式委托

UDP 中间件接口形式
``` csharp
namespace VKProxy.Middlewares;

public interface IUdpProxyMiddleware
{
    Task InitAsync(UdpConnectionContext context, CancellationToken token, UdpDelegate next); // udp没有连接建立过程，所以请求都会调用该方法

    Task<ReadOnlyMemory<byte>> OnRequestAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next); // 请求中间件，可以修改请求数据，由于协议比较底层，所以是二进制数据

    Task<ReadOnlyMemory<byte>> OnResponseAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next); // Response中间件，可以修改响应数据，由于协议比较底层，所以是二进制数据， 可能会多次调用，取决允许响应次数和实际目的端响应次数
}

public delegate Task<ReadOnlyMemory<byte>> UdpProxyDelegate(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token);

public delegate Task UdpDelegate(UdpConnectionContext context, CancellationToken token);
```

### 依赖注入接口

可以通过以下方法直接注入


``` csharp
 public static IServiceCollection UseUdpMiddleware<T>(this IServiceCollection services) where T : class, IUdpProxyMiddleware;
```

### 示例

比如我们想通过log 记录请求大小

中间件实现：

``` csharp
public class EchoUdpProxyMiddleware : IUdpProxyMiddleware
{
    private readonly ILogger<EchoUdpProxyMiddleware> logger;

    public EchoUdpProxyMiddleware(ILogger<EchoUdpProxyMiddleware> logger)
    {
        this.logger = logger;
    }

    public Task InitAsync(UdpConnectionContext context, CancellationToken token, UdpDelegate next)
    {
        return next(context, token);
    }

    public Task<ReadOnlyMemory<byte>> OnRequestAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next)
    {
        logger.LogInformation($"udp {DateTime.Now} {context.LocalEndPoint.ToString()} request size: {source.Length}");
        return next(context, source, token);
    }

    public Task<ReadOnlyMemory<byte>> OnResponseAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next)
    {
        logger.LogInformation($"udp {DateTime.Now} {context.Features.Get<IL4ReverseProxyFeature>()?.SelectedDestination?.EndPoint.ToString()} reponse size: {source.Length}");
        return next(context, source, token);
    }
}
```

注入中间件

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.UseUdpMiddleware<EchoUdpProxyMiddleware>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

## TCP 中间件

### TCP 中间件定义

由于 L4 属于比较底层的协议，本身没有类似HTTP request / response 一来必须一回的明确区分，所以不方便提供HTTP中间件那样简单的管道形式委托

TCP 中间件接口形式
``` csharp
namespace VKProxy.Middlewares;

public interface ITcpProxyMiddleware
{
    Task InitAsync(ConnectionContext context, CancellationToken token, TcpDelegate next); // tcp有连接建立过程，所以tcp初次请求会调用该方法

    Task<ReadOnlyMemory<byte>> OnRequestAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next); // 请求中间件，可以修改请求数据，由于协议比较底层，所以是二进制数据，可能会多次调用，取决实际请求次数

    Task<ReadOnlyMemory<byte>> OnResponseAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next); // Response中间件，可以修改响应数据，由于协议比较底层，所以是二进制数据， 可能会多次调用，取决实际目的端响应次数
}

public delegate Task<ReadOnlyMemory<byte>> TcpProxyDelegate(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token);

public delegate Task TcpDelegate(ConnectionContext context, CancellationToken token);
```

### 依赖注入接口

可以通过以下方法直接注入


``` csharp
 public static IServiceCollection UseTcpMiddleware<T>(this IServiceCollection services) where T : class, ITcpProxyMiddleware;
```

### 示例

比如我们想通过log 记录请求大小

中间件实现：

``` csharp
public class EchoTcpProxyMiddleware : ITcpProxyMiddleware
{
    private readonly ILogger<EchoTcpProxyMiddleware> logger;

    public EchoTcpProxyMiddleware(ILogger<EchoTcpProxyMiddleware> logger)
    {
        this.logger = logger;
    }

    public Task InitAsync(ConnectionContext context, CancellationToken token, TcpDelegate next)
    {
        return next(context, token);
    }

    public Task<ReadOnlyMemory<byte>> OnRequestAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        logger.LogInformation($"tcp {DateTime.Now} {context.LocalEndPoint.ToString()} request size: {source.Length}");
        return next(context, source, token);
    }

    public Task<ReadOnlyMemory<byte>> OnResponseAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        logger.LogInformation($"tcp {DateTime.Now} {context.Features.Get<IL4ReverseProxyFeature>()?.SelectedDestination?.EndPoint.ToString()} reponse size: {source.Length}");
        return next(context, source, token);
    }
}
```

注入中间件

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.UseTcpMiddleware<EchoTcpProxyMiddleware>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

最后还有一个socks5的示例以供大家参考[如何利用中间件扩展实现socks5](/VKProxy.Doc/docs/extensibility/socks5)