# Clusters

配置负载均衡如何处理

群集部分是命名群集的无序集合。 群集主要包含命名目标的集合及其地址，其中任何一个都被视为能够处理给定路由的请求。 代理将根据路由和群集配置处理请求，以便选择目标。

每当有多个可用的健康目标时，必须决定哪一个用于给定请求。 附带内置负载均衡算法，但也为任何自定义负载均衡方法提供扩展性。

如下则一个最简单的示例

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

首先这里是一个单节点的配置，可以极简化

``` json
{
  "ReverseProxy": {
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "Destinations": [
          {
            "Address": "127.0.0.1:5005" 
          }
        ]
      }
    }
  }
}
```

然后给大家对比一个多节点配置

``` json
{
  "ReverseProxy": {
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "LoadBalancingPolicy": "RoundRobin",
        // 这里使用 http 请求形式的主动健康检查
        "HealthCheck": {
            "Active": {
                "Enable": true,
                "Policy": "Http",
                "Path": "/test",
                "Query": "?a=d",
                "Method": "post"
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

---

Cluster同样每一个id必须唯一，不能重复，因为运行时会根据配置变动动态调整监听，不必重启实例

Cluster 有以下配置项

- `Destinations`

    Destination[] 类型

    Destination 目的配置包含以下

    - `Address`

        Url 格式 (https://google.com/search)或者 `{ip/host}:{port}` 格式 (google.com:443)

        在 http 场景由于配置可以很复杂，所以允许 Url 格式，

        不过虽然配置为 Url 格式, 但是用于 L4 时， Url 中 仅有  ip/host 和 port 会被用于转发使用

    - `Host`

        当Host 在目的端于proxy 端不一致时，可用于默认替换 Host， 比如 配置 `"Host":"cc.org"` 访问 Proxy `https://google.com/search` 会被转发为 `https://cc.org/search` 到目的端

- `LoadBalancingPolicy`

    - `Random` 

        随机选择目标。（默认值）

    - `PowerOfTwoChoices` 

        选择两个随机目标，然后选择具有最小分配请求的一个目标。 这可以避免 LeastRequests 的开销和 Random 的最坏情况，即选择一个繁忙的目标。

    - `RoundRobin`
    
        通过按顺序循环来选择目标。

    - `LeastRequests`
    
        选择分配请求最少的目标。 这需要检查所有目标。

    当上述内置策略不满足您时，您也可以自行编码扩展， 请参见 [如何扩展负载均衡策略](/VKProxy.Doc/docs/extensibility/loadbalancingpolicy)

- `HealthCheck`

    在大多数实际系统中，其节点偶尔会遇到暂时性问题，并完全由于各种原因（例如过载、资源泄漏、硬件故障等）而出现故障。理想情况下，最好完全防止这些不幸事件以主动方式发生，但设计和构建这种理想系统的成本一般高。 但是，还有另一种更便宜的反应方法，旨在最大程度地减少由于失败而导致的负面影响对客户端请求的影响。 代理可以分析每个节点的运行状况，并停止将客户端流量发送到不正常的节点，直到它们恢复为止。 KVProxy 以主动和被动目标地址健康检查的形式实现该方法。 用户可以根据自己情况选择使用或者不使用。 运行状况状态使用 Unknown 值进行初始化，稍后可以通过相应的策略更改为 Healthy 或 Unhealthy。

    - `Active`

        可以通过向指定的健康终结点发送定期探查请求并分析响应的结果，主动监视目标服务器的健康状况。 该分析根据为集群指定的主动健康检查策略执行，并计算出新的目标健康状态。 最后，策略根据 HTTP 响应代码（2xx 被视为正常）将每个目标标记为正常或不正常，并重新生成群集的正常目标集合。

        - `Enabled`
            
            指示是否为群集启用主动运行状况检查的标志

        - `Interval`

            发送健康探测请求的间隔。 默认 00:01:00
        
        - `Timeout`

            探测请求超时。 默认 00:00:10

        - `Policy`
        
            用于评估目标的主动运行状况状态的策略名称。 必需参数, 内置如下两种策略 （同理策略也是可以扩展的）
            - `Connect`

                仅仅测试tcp端口是否能连接，不做任何处理

                此策略 `Path`/ `Query`/`Method` 无效

            - `Http`

                通过http 检查响应代码（2xx 被视为正常）

        - `Path`
        
            所有群集目标上的运行状况检查路径。 默认 null。

        - `Query`
        
            所有群集目标上的运行状况检查查询。 默认 null。

        - `Method`
        
            所有群集目标上的运行状况检查查询Method。 默认 GET。

        - `Passes`
        
            认定目标恢复正常所需达到成功次数。 默认 1

        - `Fails`
        
            认定目标变成不正常所需达到失败次数。 默认 1

    - `Passive`

        可以被动监视客户端请求代理中的成功和失败，以响应性评估目标运行状况。 对代理请求的响应被专用被动运行状况检查中间件截获，该中间件将其传递给群集上配置的策略。 该政策分析响应，以评估产生它们的目的地是否健康。 然后，它会计算新的被动运行状况状态并将其分配给相应的目标，并重新生成群集的正常目标集合。

        请注意，响应通常在被动运行状况策略运行之前发送到客户端，因此策略无法截获响应正文，也无法修改响应标头中的任何内容，除非代理应用程序引入了完整的响应缓冲。

        与主动运行状况检查逻辑有一个重要区别。 一旦某个目标被指定为不健康的被动状态，它就会停止接收所有新流量，从而阻碍未来的健康状况重新评估。 该策略还计划在配置的期限后重新激活目标。 重新激活意味着将被动运行状况状态从 Unhealthy 重置回初始 Unknown 值，从而使目标再次符合流量条件。

        - `Enabled` 
        
            指示是否为群集启用被动运行状况检查的标志。 

        - `ReactivationPeriod` 
        
            经过一段设定的时间后，状态异常的目标服务器的被动健康状态将重置为 Unknown，并开始重新接收流量。 默认值为  00:01:00
            
        - `DetectionWindowSize` 
        
            在速率计算中保留并考虑检测到的故障的时间段。 默认值为 00:01:00 
            
        - `MinimalTotalCountThreshold` 
        
            在该策略开始评估目标的运行状况并强制执行失败率限制之前，必须代理到检测窗口内目标的最少请求总数。 默认值为 10。 
            
        - `FailureRateLimit` 
        
            未在群集元数据上设置时应用的标记为运行不正常的目标的默认故障率限制。 该值在 (0,1)范围内。 默认值为 0.3（30%）。 