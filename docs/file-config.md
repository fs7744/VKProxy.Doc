# 在`appsettings.json`中配置

VKProxy 可以从 `IConfiguration` 实例加载配置。 默认情况下，配置从 `ReverseProxy` 部分加载

其下可配置如下内容

- [ReverseProxyOptions](/docs/file-config/options) 路由相关参数
- [Listen](/docs/file-config/listen)  监听配置
- [Routes](/docs/file-config/route)  路由配置
- [Clusters](/docs/file-config/cluster)  负载均衡配置
- [Sni](/docs/file-config/sni)  证书相关配置


### 修改默认配置项

如需切换配置项，可采取以下方式

# [自行构建](#tab/build)

可以通过修改`ReverseProxyOptions.Section`调整配置项

例如下面切换使用`TextSection`配置项

```csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        // 需注意 必须在 UseReverseProxy() 之前，因为 UseReverseProxy() 已经有相关配置处理，不在之前配置，会导致部分配置无法正确加载
        i.Configure<ReverseProxyOptions>(o => o.Section = "TextSection");
    })
    .UseReverseProxy()
    .Build();
```

`ReverseProxyOptions` 还可以配置相关项以调整程序性能，具体可参见[服务器参数](/docs/file-config/options)

