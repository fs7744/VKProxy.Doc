# 不同监听场景如何配置

### 监听 UDP

这里展示如何 将 `127.0.0.1:5000` UDP 代理到 `127.0.0.1:11000`, 同时只接受服务端最多返回一个 udp 包

# [appsettings.json](#tab/udp-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : udptest
      "udptest": {
        "Protocols": [ "UDP" ], // 协议选择 UDP
        "Address": [ "127.0.0.1:5000" ], // 监听地址
        "RouteId": "udpTestRoute"  // UDP 必填路由，因为不像http 有host等协议参数能确定目的地址
      }
    },
    "Routes": {
    // Routes id : udpTestRoute
      "udpTestRoute": {
        "ClusterId": "udpTestCluster",  // udp 目的地址 id
        "UdpResponses": 1,  // 只接受服务端最多返回一个 udp 包
        "Timeout": "00:00:11"  // 超时时间 11 秒， 包含从udp包 发送到 目的地址和 等待以及返回 服务端udp包的总共时间
      }
    },
    "Clusters": {
        // Clusters id : udpTestCluster
      "udpTestCluster": {
        "Destinations": [
          {
            "Address": "127.0.0.1:11000"  //目的地址
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/udp-ui)

暂未支持

---

