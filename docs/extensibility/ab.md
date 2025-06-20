# A/B 测试和滚动升级

这里简单介绍一下A/B 测试和滚动升级相关场景在Api Gateway (比如VKProxy) 可以怎么做

## A/B 测试

A/B 测试（Split Testing）是一种对比实验方法，用于评估两个（或多个）版本之间的效果差异。其核心思想是将用户流量随机分为两组（或多组），分别体验不同版本（A、B、C...），然后通过关键指标（如转化率、点击率、留存率等）的变化，判断哪个版本更优。

**A/B 测试的流程**：

- 确定目标（如提升注册率、减少跳出率）。
- 设计对比版本（A为原始版本，B为新版本）。
- 随机将用户分配到不同组（常用 cookie、用户 ID、灰度发布工具等实现）。
- 收集和分析数据，统计每组指标。

得出结论，决定是否全面上线新版本。

### 通过路由区分A/B

在大部分场景，路由足以满足大家区分A/B环境

举例 app新增或者调整了的一些功能，处于稳妥考虑，领导希望优先找一小部分用户体验测试，收集反馈在全面发布前看看是否做一些调整。负责后台api的你采取了简单的 A/B 策略， 要求 测试app 调用后台api时加一个 header `x-env: test`, 你调整路由通过此header区分允许测试app 访问尚未正式发布的部分api

配置示例：

``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,  // 优先级最高，优先尝试匹配 路由 a
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ],
            "Statement": "Header('x-env') = 'test'"
        },
        "ClusterId": "ClusterA",
      },
      // 通过降低优先级，路由 a 不匹配的请求会走入路由 b
      "b": {
        "Order": 1, 
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "ClusterB",
      }
    },
    "Clusters": {
        "ClusterA": {
            "Destinations": [
                {
                    "Address": "http://127.0.0.1:7930"
                }
            ]
        },
        "ClusterB": {
            "Destinations": [
                {
                    "Address": "http://127.0.0.2:8989"
                }
            ]
        }
    }
  }
}
```

当然路由区分还可以编写其他复杂条件，可以参见 [如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)

您还可以通过[定制化扩展](/VKProxy.Doc/docs/extensibility) 发挥创造力，定制化自己的复杂规则

## 滚动升级

滚动升级（Rolling Update）是一种逐步替换旧版本为新版本的发布策略，常用于后端服务、微服务、容器集群（如 Kubernetes）、云平台等。其特点是“分批次、逐步替换”，这样可以保证服务持续可用，降低因上线新版本导致的风险。

**滚动升级的流程**：

- 将部分实例（如1台、10%节点）升级为新版本，其余保持旧版本运行。
- 观察新版本运行状况（如健康检查、监控指标、错误率等）。
- 如果新版本无异常，则继续升级下一批，直至全部替换完成。
- 若新版本出现异常，可快速回滚，影响面有限。

VKProxy 由于支持运行时配置动态变更，所以可以说通常简单场景的滚动升级

目前暂未集成Kubernetes实现

配置修改api目前只有 支持etcd的UI站点有简单的api

不过可以通过配置变动向大家说明一下通常滚动升级场景配置是如何变动的

假设您有如下 api 配置

``` json
{
  "ReverseProxy": {
    "Routes": {
      "b": {
        "Order": 0, 
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "ClusterB",
      }
    },
    "Clusters": {
        "ClusterB": {
            "LoadBalancingPolicy": "RoundRobin",
            "HealthCheck": {
                "Active": {
                    "Enable": true,
                    "Interval": "00:00:30",
                    "Policy": "Http",
                    "Path": "/health"  // 业务实例暴露有健康检查api， 正常运行会返回 200 ，当下线时会返回 500
                }
            },
            "Destinations": [
                {
                    "Address": "http://127.0.0.2:8989"
                }
            ]
        }
    }
  }
}
```

当有新版本上线您可以通过类似配置修改达到滚动升级的目的  （已集成Kubernetes的api gateway 也可以干的类似事情，不过这样滚动升级策略会存在一定时间内新旧版本同时提供服务的场景，需要结合业务场景取舍）

首先部署了一部分新实例， 比如 127.0.0.3:8080

然后会将这一部分实例加入 api 配置中 （Kubernetes之类工具通常还会有检查实例正常启动之后再处理的过程）

``` json
{
  "ReverseProxy": {
    "Clusters": {
        "ClusterB": {
            "Destinations": [
                {
                    "Address": "http://127.0.0.2:8989"
                },
                {
                    "Address": "http://127.0.0.3:8080"
                }
            ]
        }
    }
  }
}
```

接着您检查了一段时间监控，一切运行正常，觉得可以下掉旧实例了

为了稳妥，您并未直接删除127.0.0.2:8989实例， 而是先通过健康检查下线了旧实例 （让 http://127.0.0.2:8989/health 返回 500）

观察一段时间，127.0.0.2:8989 没有任何流量了，您再修改了配置，删除了实例

``` json
{
  "ReverseProxy": {
    "Clusters": {
        "ClusterB": {
            "Destinations": [
                {
                    "Address": "http://127.0.0.3:8080"
                }
            ]
        }
    }
  }
}
```

多实例场景就是反复执行上述行为，

如遇见新版本实例有问题，就会撤下新实例，恢复旧实例配置，由于滚动进行，出现问题时通常只有部分实例受影响，所以是有效保证上线稳定性的一种策略

ps： 下线旧实例这一步，Kubernetes之类工具还有另外一种简单做法，先从api gateway之类配置移除旧实例，但旧实例不立马删除，而是等待一定时间（足够保证没有访问流量）再直接删除

大家可以根据自己所需实施具体的滚动升级或金丝雀之类策略

当然您还可以通过[定制化扩展](/VKProxy.Doc/docs/extensibility) 发挥创造力，定制化自己的复杂规则