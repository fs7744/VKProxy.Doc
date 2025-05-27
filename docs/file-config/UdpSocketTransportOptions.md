# UdpSocketTransportOptions

简单实现的 udp 支持对应配置

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

``` json
{
  "ServerOptions": {
    "Socket": {
        "UdpPoolSize": 1024
    }
  },
}
```

# [code](#tab/code)

通过`代码`配置

``` csharp
.ConfigureServices(i =>
{
    i.Configure<UdpSocketTransportOptions>(o => o.UdpPoolSize = 1024);
})
```

---

- `UdpMaxSize`

    int 类型, 默认值 4096

    udp 包最大大小（也是处理udp buffer大小，所以过大会非常影响内存和性能，网络通信上通常不支持过大的udp包）

- `UdpPoolSize`

    int 类型, 默认值 1024

    udp处理器复用池大小