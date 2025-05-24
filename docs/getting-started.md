# 入门

## 安装

# [自行构建](#tab/build)

#### 创建一个项目，然后选取所需package安装

``` shell
dotnet add package VKProxy
dotnet add package VKProxy.Storages.Etcd
```

#### 配置程序

``` csharp
using Microsoft.Extensions.Hosting;
using ProxyDemo;

var app = Host.CreateDefaultBuilder(args)
    .UseReverseProxy()
    .ConfigureServices(i =>
    {
        // 默认加载appsettings.json 配置
        // 如需采用 etcd 作为配置源，请设置环境变量`ETCD_CONNECTION_STRING`并使用如下代码
        // i.UseEtcdConfigFromEnv();
    })
    .Build();

await app.RunAsync();
```

#### 配置代理

假如使用 `appsettings.json` 进行配置, 这里列举一个http1的代理配置

(配置项很多，可参考后续[具体配置项说明](/docs/file-config)， 也可以考虑使用 [UI配置站点](/docs/ui-config))

``` json
{
  "ReverseProxy": {
    // 监听端口 127.0.0.1:5001
    "Listen": {
      "http": {
        "Protocols": [
          "Http1"
        ],
        "Address": [
          "127.0.0.1:5001"
        ]
      }
    },
    // 设置路由匹配任意 com 结尾的host，以及 url开头为 /ws 并且 method 为 GET的请求
    "Routes": {
      "HTTPTEST": {
        "Match": {
          "Hosts": [
            "*com"
          ],
          "Paths": [
            "/ws*"
          ],
          "Statement": "Method = 'GET'"
        },
        "ClusterId": "apidemo",
        "Timeout": "00:10:11"
      }
    },
    // 配置匹配请求转发到对应目标地址，并且通过主动健康检查保证服务稳定
    "Clusters": {
      "apidemo": {
        "LoadBalancingPolicy": "RoundRobin",
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
            "Address": "http://127.0.0.1:1104"
          },
          {
            "Address": "https://google.com"
          }
        ]
      }
    }
  }
}
```

# [Docker](#tab/Docker)

暂不支持，还没时间搞到这一步
