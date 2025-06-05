# 定制化扩展

为了方便大家使用 KVProxy 在一些场景，默认功能无法满足时，可以通过自定义扩展实现自己的需求。

同时也是遵照 asp.net core 设计理念，提供了两种扩展方式

## 中间件管道

中间件是一种装配到应用管道以处理请求和响应的软件。 每个组件：

- 选择是否将请求传递到管道中的下一个组件。
- 可在管道中的下一个组件前后执行工作。

请求委托用于生成请求管道。 请求委托处理每个 HTTP/tcp/udp 请求。

具体概念可参考[ASP.NET Core 中间件](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/middleware/?view=aspnetcore-9.0)

![](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/servers/yarp/media/yarp-pipeline.png?view=aspnetcore-9.0)

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


## ReverseProxyFeature

除了两大扩展方式之外，还有一个接口数据在运行时有表明当前路由匹配情况

``` csharp
public interface IReverseProxyFeature  // http 路由会使用该接口
{
    public RouteConfig Route { get; set; } // 匹配上的路由，如为 null 则未匹配任何路由
    public DestinationState? SelectedDestination { get; set; } // 在选中健康的目标地址后，对应配置会设置在这里
}

public interface IL4ReverseProxyFeature : IReverseProxyFeature // tcp / udp 路由会使用该接口
{
    public bool IsDone { get; set; }  // 表明是否已经处理，当为 true 时，KVProxy 内置L4代理将不会进行代理
    public bool IsSni { get; set; }   // 表明是否为 tcp sni 代理模式
    public SniConfig? SelectedSni { get; set; }  // 为 tcp sni 代理模式时的配置
}
```

运行时可通过 feature 获取， 比如

``` csharp
// http
internal class EchoHttpMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var f = context.Features.Get<IReverseProxyFeature>();
    }
}

//tcp
internal class EchoTcpProxyMiddleware : ITcpProxyMiddleware
{
    public Task InitAsync(ConnectionContext context, CancellationToken token, TcpDelegate next)
    {
        var f = context.Features.Get<IL4ReverseProxyFeature>();
    }

    public Task<ReadOnlyMemory<byte>> OnRequestAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        var f = context.Features.Get<IL4ReverseProxyFeature>();
    }

    public Task<ReadOnlyMemory<byte>> OnResponseAsync(ConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, TcpProxyDelegate next)
    {
        logger.LogInformation($"tcp {DateTime.Now} {context.Features.Get<IL4ReverseProxyFeature>()?.SelectedDestination?.EndPoint.ToString()} reponse size: {source.Length}");
    }
}

//udp
internal class EchoUdpProxyMiddleware : IUdpProxyMiddleware
{
    public Task InitAsync(UdpConnectionContext context, CancellationToken token, UdpDelegate next)
    {
        var f = context.Features.Get<IL4ReverseProxyFeature>();
    }

    public Task<ReadOnlyMemory<byte>> OnRequestAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next)
    {
        var f = context.Features.Get<IL4ReverseProxyFeature>();
    }

    public Task<ReadOnlyMemory<byte>> OnResponseAsync(UdpConnectionContext context, ReadOnlyMemory<byte> source, CancellationToken token, UdpProxyDelegate next)
    {
        logger.LogInformation($"udp {DateTime.Now} {context.Features.Get<IL4ReverseProxyFeature>()?.SelectedDestination?.EndPoint.ToString()} reponse size: {source.Length}");
    }
}
```

不建议大家直接修改 `IReverseProxyFeature` 的值，可能会破坏路由

## 可扩展套接字应用程序框架

除了代理功能外，由于通过反射释放了Kestrel的能力，你也可以把 VKProxy 当成可扩展套接字应用程序框架使用

使用它轻松构建始终连接的套接字应用程序，而无需考虑如何使用套接字，如何维护套接字连接以及套接字如何工作。

(在Kestrel基础上开发，理论可以帮大家节省一些比如直接socket要自己管理 socket之类的事情)

具体可以参考[可扩展套接字应用程序框架](/VKProxy.Doc/docs/extensibility/socket)