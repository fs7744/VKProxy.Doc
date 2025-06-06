# 如何扩展负载均衡策略

已经内置如下负载均衡策略

- `Random` 

    随机选择目标。（默认值）

- `PowerOfTwoChoices` 

    选择两个随机目标，然后选择具有最小分配请求的一个目标。 这可以避免 LeastRequests 的开销和 Random 的最坏情况，即选择一个繁忙的目标。

- `RoundRobin`

    通过按顺序循环来选择目标。

- `LeastRequests`

    选择分配请求最少的目标。 这需要检查所有目标。

如果不满足大家需求，可以通过如下方式扩展

首先实现`ILoadBalancingPolicy`接口

```csharp
public interface ILoadBalancingPolicy
{
    string Name { get; }

    DestinationState? PickDestination(IReverseProxyFeature feature, IReadOnlyList<DestinationState> availableDestinations);
}
```

这里直接列举 最简单的`Random` 实现


```csharp
public sealed class TestRandomLoadBalancingPolicy : ILoadBalancingPolicy
{
    public string Name => "TestRandom";

    public DestinationState? PickDestination(IReverseProxyFeature feature, IReadOnlyList<DestinationState> availableDestinations)
    {
        return availableDestinations[Random.Shared.Next(availableDestinations.Count)];
    }
}
```

然后注入DI

``` csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices(i =>
    {
        i.AddSingleton<ILoadBalancingPolicy, TestRandomLoadBalancingPolicy>(); // 这一行
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
        "LoadBalancingPolicy": "TestRandom", // 使用自定义的策略
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