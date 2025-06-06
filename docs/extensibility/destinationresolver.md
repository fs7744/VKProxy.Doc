# 如何扩展服务发现

使用目标解析程序来扩展已配置的目标地址集。 目标解析程序可用作与服务发现系统的集成点。

只需实现以下接口

``` csharp
public interface IDestinationResolver
{
    int Order { get; } // 执行优先级，数字越小优先级越高

    Task<IDestinationResolverState> ResolveDestinationsAsync(ClusterConfig cluster, List<DestinationConfig> destinationConfigs, CancellationToken cancellationToken);  
}

public sealed record DestinationConfig
{ 
    public string Address { get; init; } = default!;

    public string? Host { get; init; }
}
```

不过`IDestinationResolverState` 不是单纯静态数组，而是允许动态变动的抽象

如果不需要动态，你可以返回静态数组

而比如 dns 服务发现等场景，您可以根据需要随时刷新变化 

``` csharp
public interface IDestinationResolverState : IReadOnlyList<DestinationState>, IDisposable
{
}
```

这里列举 dns 实现来说明或许会容易点

## 静态 dns

这里我们首先完成一个简单的静态 dns，只再第一次查询dns，解析ip，而后不再定时检查刷新

``` csharp
using System.Net;
using VKProxy.Config;
using VKProxy.ServiceDiscovery;

namespace ProxyDemo.IDestinationResolvers;

public class StaticDNS : IDestinationResolver
{
    public int Order => -1;// 确保比默认dns 优先级高

    public static async Task<IEnumerable<DestinationState>> QueryDNSAsync(ClusterConfig cluster, DestinationConfig destinationConfig, CancellationToken cancellationToken)
    {
        cancellationToken.ThrowIfCancellationRequested();

        // 由于address 是 uri 格式，所以需要解析 host
        var originalUri = new Uri(destinationConfig.Address);
        var originalHost = destinationConfig.Host is { Length: > 0 } host ? host : originalUri.Authority;
        var hostName = originalUri.DnsSafeHost;

        // dns query
        var addresses = await Dns.GetHostAddressesAsync(hostName, cancellationToken).ConfigureAwait(false);

        // 修改 uri
        var uriBuilder = new UriBuilder(originalUri);
        return addresses.Select(i =>
        {
            // 修改 host 为 ip
            var addressString = i.ToString();
            uriBuilder.Host = addressString;

            return new DestinationState()
            {
                EndPoint = new IPEndPoint(i, originalUri.Port), // 设置ip ，提供给 非http 场景使用
                ClusterConfig = cluster,  // cluster 涉及健康检查配置，所以需要赋值
                Host = originalHost,  // 设置原始host，避免 http 访问失败
                Address = uriBuilder.Uri.ToString() // 设置修改后的 uri
            };
        });
    }

    public async Task<IDestinationResolverState> ResolveDestinationsAsync(ClusterConfig cluster, List<DestinationConfig> destinationConfigs, CancellationToken cancellationToken)
    {
        var tasks = destinationConfigs.Select(async i => await QueryDNSAsync(cluster, i, cancellationToken));
        await Task.WhenAll(tasks);

        return new StaticDestinationResolverState(tasks.SelectMany(i => i.Result).ToArray());
    }
}
```

配置

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<IDestinationResolver, StaticDNS>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

## 动态 dns

这里我们再完成一个简单的动态 dns，不止第一次查询dns，解析ip，而后定时检查刷新检查是否变化


``` csharp
using System.Net;
using VKProxy.Config;
using VKProxy.ServiceDiscovery;

namespace ProxyDemo.IDestinationResolvers;

namespace ProxyDemo.IDestinationResolvers;

public class NonStaticDNS : DestinationResolverBase // 基于 DestinationResolverBase 可以简化重复编码，cluster destinationConfigs 都会被放在 FuncDestinationResolverState 中持久
{
    public override int Order => -2;

    public override async Task ResolveAsync(FuncDestinationResolverState state, CancellationToken cancellationToken)
    {
        var tasks = state.Configs.Select(async i => await StaticDNS.QueryDNSAsync(state.Cluster, i, cancellationToken));
        await Task.WhenAll(tasks);
        state.Destinations = tasks.SelectMany(i => i.Result).ToArray(); // 变更只需要赋值替换就好， 您还可以加入变更检查，在数据未变化时减少替换的影响

        // 这里用简单的 CancellationChangeToken 延迟触发变更
        var cts = new CancellationTokenSource();
        cts.CancelAfter(60000);

        new CancellationChangeToken(cts.Token).RegisterChangeCallback(o =>
        {
            if (o is FuncDestinationResolverState s)
            {
                // 只需再次调用 ResolveAsync
                ResolveAsync(s, new CancellationTokenSource(60000).Token).ConfigureAwait(false).GetAwaiter().GetResult();
            }
        }, state);
    }
}
```

配置

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<IDestinationResolver, NonStaticDNS>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```