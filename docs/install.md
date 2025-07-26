# 安装

# [dotnet tool](#tab/tool)

提供简单的命令行工具，可以在本地进行相关测试

``` shell
dotnet tool install --global VKProxy.Cli
```

不过目前只支持 net9.0 (net10 正式发布后会切换制net10)

安装后可以使用如下命令
``` shell
vkproxy -h
// it will output
--help (-h)    show options, View more at https://fs7744.github.io/VKProxy.Doc/docs/introduction.html
--version    VKProxy version
proxy     L4/L7 proxy build on Kestrel
     --help (-h)
         show options
     --config (-c,Environment:VKPROXY_CONFIG)
         json file config, like /xx/app.json
     --etcd (Environment:ETCD_CONNECTION_STRING)
         etcd address, like http://127.0.0.1:2379
     --etcd-prefix (Environment:ETCD_PREFIX)
         default is /ReverseProxy/
     --etcd-delay (Environment:ETCD_DELAY)
         delay change config when etcd change, default is 00:00:01
     --socks5 (Environment:VKPROXY_SOCKS5)
         use simple socks5 support
     --sampler (Environment:VKPROXY_SAMPLER)
         log sampling, support trace/random/none
     --memory-cache-max (Environment:VKPROXY_MEMORY_CACHE_MAX)
         Memory Cache Size Limit
     --memory-cache-percentage (Environment:VKPROXY_MEMORY_CACHE_COMPACTION_PERCENTAGE)
         Memory Cache Compaction Percentage
     --disk-cache (Environment:VKPROXY_DISK_CACHE)
         disk cache directory
     --disk-cache-max (Environment:VKPROXY_DISK_CACHE_MAX)
         disk cache Size Limit
     --redis (Environment:VKPROXY_REDIS)
         StackExchangeRedis config
     --redis-data-protection (Environment:VKPROXY_REDIS_DATA_PROTECTION)
         DataProtection sotre in redis key
     --redis-pool-size (Environment:VKPROXY_REDIS_POOL_SIZE)
         StackExchangeRedis pool size, default is 10
acme     cli tool to help you issues certificates through an automated API based on the ACME protocol.
     terms     view current terms of service.
     account     Manage ACME accounts.
     order     Manage ACME orders.
```

#### 如果使用json文件配置

配置项很多，可参考后续[具体配置项说明](/VKProxy.Doc/docs/file-config)

这里举个例子 

创建json文件

``` json
{
  "ReverseProxy": {
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

然后启动

``` shell
vkproxy proxy -c D:\code\test\proxy\config.json

// 启动后会看到类似如下的内容
info: VKProxy.Server.ReverseProxy[3]
      Listening on: [Key: http,Protocols: HTTP1,EndPoint: 127.0.0.1:5001]
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: D:\code\test\proxy
warn: VKProxy.Server.ReverseProxy[5]
      Active health failed, can not connect socket 127.0.0.1:1104 No connection could be made because the target machine actively refused it. (127.0.0.1:1104).
```

#### 使用 etcd 配置

在多实例的情况，同一份配置分发就比较麻烦， 这里提供 ui 可以配置etcd + agent 从etcd读取配置 方便大家使用

ui使用可以参考 [UI配置站点](/VKProxy.Doc/docs/ui-config)

用tool 启动 agent 可以这样使用

``` shell
vkproxy proxy --etcd http://127.0.0.1:2379 --etcd-prefix /ReverseProxy/

// 启动后会看到类似如下的内容
info: VKProxy.Server.ReverseProxy[3]
      Listening on: [Key: http,Protocols: HTTP1,EndPoint: 127.0.0.1:5001]
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: D:\code\test\proxy
warn: VKProxy.Server.ReverseProxy[5]
      Active health failed, can not connect socket 127.0.0.1:1104 No connection could be made because the target machine actively refused it. (127.0.0.1:1104).
```


# [Docker](#tab/Docker)

当大家基本代理功能足够时，简化大家使用成本/快速构建的默认已构建镜像

所有的镜像可以在 [docker hub vkproxy agent](https://hub.docker.com/r/vkproxy/agent) 找到

提供如下环境变量

- VKPROXY_CONFIG
  
  json file config, like /xx/app.json

  example `VKPROXY_CONFIG=/xx/app.json`
- VKPROXY_SOCKS5

  use simple socks5 support

  example `VKPROXY_SOCKS5=true`

- ETCD_CONNECTION_STRING

  etcd address, like http://127.0.0.1:2379

  example `ETCD_CONNECTION_STRING=http://127.0.0.1:2379`

- ETCD_PREFIX

  default is /ReverseProxy/

  example `ETCD_PREFIX=/ReverseProxy/`

- ETCD_DELAY
  
  delay change config when etcd change, default is 00:00:01

- VKPROXY_MEMORY_CACHE_MAX
  
  Memory Cache Size Limit

- VKPROXY_MEMORY_CACHE_COMPACTION_PERCENTAGE
  
  Memory Cache Compaction Percentage

- VKPROXY_REDIS_POOL_SIZE
  
  StackExchangeRedis pool size, default is 10

- VKPROXY_REDIS
  
  StackExchangeRedis config

- VKPROXY_REDIS_DATA_PROTECTION
  
  DataProtection sotre in redis key

- VKPROXY_DISK_CACHE
  
  disk cache directory

- VKPROXY_DISK_CACHE_MAX
  
  disk cache Size Limit

#### 如果使用json文件配置

配置项很多，可参考后续[具体配置项说明](/VKProxy.Doc/docs/file-config)

这里举个例子 

创建json文件

``` json
{
  "ReverseProxy": {
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

然后启动

``` shell
docker run --rm -v /mnt/d/code/test/proxy:/config -e VKPROXY_CONFIG=/config/config
.json -e ETCD_CONNECTION_STRING= --network host a.newegg.org/docker-hub-remote/vkproxy/agent:0.0.0.7

// 启动后会看到类似如下的内容
info: VKProxy.Server.ReverseProxy[3]
      Listening on: [Key: http,Protocols: HTTP1,EndPoint: 127.0.0.1:5001]
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
warn: VKProxy.Server.ReverseProxy[5]
      Active health failed, can not connect socket [2404:6800:4012:7::200e]:443 Network is unreachable ([2404:6800:4012:7::200e]:443).
warn: VKProxy.Server.ReverseProxy[5]
      Active health failed, can not connect socket 127.0.0.1:1104 Connection refused (127.0.0.1:1104).
```

#### 使用 etcd 配置

在多实例的情况，同一份配置分发就比较麻烦， 这里提供 ui 可以配置etcd + agent 从etcd读取配置 方便大家使用

ui使用可以参考 [UI配置站点](/VKProxy.Doc/docs/ui-config)

用 docker 启动 agent 可以这样使用

``` shell
docker run --rm -e ETCD_CONNECTION_STRING=http://127.0.0.1:2379 --network host vkproxy/agent:0.0.0.7

// 启动后会看到类似如下的内容
info: VKProxy.Server.ReverseProxy[3]
      Listening on: [Key: http,Protocols: HTTP1,EndPoint: 127.0.0.1:5001]
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: D:\code\test\proxy
warn: VKProxy.Server.ReverseProxy[5]
      Active health failed, can not connect socket 127.0.0.1:1104 No connection could be made because the target machine actively refused it. (127.0.0.1:1104).
```


# [自行构建](#tab/build)

VKProxy 有很多扩展点可以定制化大家所需特制化需求，所以有相关需求时推荐自行构建

#### 创建一个项目，然后选取所需package安装

``` shell
dotnet add package VKProxy.Host
```

#### 配置程序

``` csharp
using Microsoft.Extensions.Hosting;
using ProxyDemo;

var app = VKProxyHost.CreateBuilder()
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

(配置项很多，可参考后续[具体配置项说明](/VKProxy.Doc/docs/file-config)， 也可以考虑使用 [UI配置站点](/VKProxy.Doc/docs/ui-config))

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
