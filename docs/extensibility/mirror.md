# http流量镜像

“流量镜像”是指将网络中的数据流量复制一份，并将这份复制流量发送到另一个目的地（如监控、分析或安全检测系统）。这项技术常用于网络安全、故障排查、业务灰度发布等场景。

主要应用场景
- 安全监控与威胁检测
    
    将生产环境的流量镜像到安全分析设备（如IDS/IPS），用于实时监控和威胁检测。

- 性能分析与故障排查

    将流量镜像到分析平台，对网络异常、延迟、丢包等问题进行实时排查和定位。

-   灰度发布和A/B测试
    
    将真实用户流量镜像到新版本服务，进行灰度环境验证和兼容性测试，不影响真实用户体验。

- 合规与审计

    对重要业务流量进行行为记录，以满足合规和审计要求。

VKProxy 目前只支持 http流量镜像, 需注意由于会再一次发送http 请求，请求body会临时暂存内存，所以无论内存还是请求延迟都会受到影响，特别body很大的请求

## 设置

大家可以在`Metadata`中设置缓存， 具体设置项如下

- `MirrorCluster`

    镜像流量发送到的集群id


配置示例：

``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "apidemo",
        "Metadata": {
          "MirrorCluster": "apidemoMirror"
        }
      }
    },
    "Clusters": {
      "apidemo": {
        "LoadBalancingPolicy": "Hash",
        "Metadata": {
          "HashBy": "header",
          "Key": "X-forwarded-For"
        },
        "Destinations": [
          {
            "Address": "https://xxx.lt"
          }
        ]
      },
      "apidemoMirror": {
        "LoadBalancingPolicy": "Hash",
        "Metadata": {
          "HashBy": "header",
          "Key": "X-forwarded-For"
        },
        "Destinations": [
          {
            "Address": "http://xxx.org/"
          }
        ]
      }
    }
  }
}
```