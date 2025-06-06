# 如何扩展主动健康检查策略

已经内置如下主动健康检查策略

- `Connect`

    仅仅测试tcp端口是否能连接，不做任何处理

    此策略 `Path`/ `Query`/`Method` 无效

- `Http`

    通过http 检查响应代码（2xx 被视为正常）

如果不满足大家需求，可以通过如下方式扩展

首先实现`IActiveHealthChecker`接口

``` csharp
public interface IActiveHealthChecker
{
    string Name { get; }

    Task CheckAsync(ActiveHealthCheckConfig config, DestinationState state, CancellationToken cancellationToken);
}
```

如果需要简单的 失败/成功 的计数处理可以基于 `ActiveHealthCheckerBase`, 


``` csharp
public abstract class ActiveHealthCheckerBase : IActiveHealthChecker
{
    public abstract string Name { get; }

    protected abstract ValueTask<bool> DoCheckAsync(ActiveHealthCheckConfig config, DestinationState state, CancellationToken cancellationToken);
}
```

这里用最简单的 tcp connect 作为举例


``` csharp
public class TestConnectionActiveHealthChecker : ActiveHealthCheckerBase
{
    private readonly IConnectionFactory connectionFactory;

    public override string Name => "TestConnect";

    public TestConnectionActiveHealthChecker(IConnectionFactory connectionFactory, ProxyLogger logger) : base(logger)
    {
        this.connectionFactory = connectionFactory;
    }

    protected override async ValueTask<bool> DoCheckAsync(ActiveHealthCheckConfig config, DestinationState state, CancellationToken cancellationToken)
    {
        var c = await connectionFactory.ConnectAsync(state.EndPoint, cancellationToken);  // 就 tcp connect 一下， 成功 返回true， 失败这里直接抛异常
        c.Abort();
        return true;
    }
}
```


然后注入DI

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<IActiveHealthChecker, TestConnectionActiveHealthChecker>(); // 这一行
    })
    .UseReverseProxy()
    .Build();

await app.RunAsync();
```

大家在配置时还要指定一下


``` json
{
  "ReverseProxy": {
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "HealthCheck": {
            "Active": {
                "Enable": true,
                "Policy": "TestConnect"  // 使用自定义的策略
            }
        },
        "Destinations": [
          {
            "Address": "127.0.0.1:5005" 
          },
          {
            "Address": "198.0.0.1:5005" 
          },
          {
            "Address": "198.0.0.2:5005" 
          }
        ]
      }
    }
  }
}
```