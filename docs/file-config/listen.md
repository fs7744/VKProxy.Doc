# Listen

> [!WARNING]
> 虽然可以在运行时动态变更监听，但是目前[Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel)在终止监听时会等待已有连接结束，以降低对用户影响，达到优雅关闭的效果
>
> 所以在动态变更时不一定能实时生效，一定会等待已访问连接结束(超时时间1秒，超时将强制关闭)，如果频繁变化，可能会导致问题

终结点侦听传入l4/l7连接。 创建终结点时，必须具有唯一的id并且使用它将侦听的地址对其进行配置。 通常，这是一个 IP 地址和端口号。 比如下面 id 为 http 的 一个listen 配置 

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

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
    }
  }
}
```

---

支持多个不同listen 同时启用，但每一个id必须唯一，不能重复，因为运行时会根据配置变动动态调整监听，不必重启实例

如下列举具体配置配置项 (各种不同监听配置场景请参见[不同监听场景如何配置](/VKProxy.Doc/docs/howtolisten), 这里只列举listen 相关配置，无法具体说明)

listen 有以下配置项

- `Protocols`

    监听端口将进行何种协议处理，支持如下

    - `UDP`

        UDP 代理， 必须同时配置 `RouteId` 以明确代理到哪一个目的地址

    - `TCP`

        TCP 代理 支持多种模式
        - sni 代理模式 
        
            即通过 ssl 证书握手时的 host 决定代理到哪一个目的地址的方式,
            
            访问请求也必须 ssl 加密，但是该模式可以复用端口，已节约资源，
            
            比如 443 端口可以同时服务 `https://a.com` 和 `https://test.org`

            必须配置 `UseSni = true` 

            当然证书握手不一定在 proxy 端处理，也可以在目的端才进行握手校验，具体配置请参见 [SNI](/VKProxy.Doc/docs/sni)

        - 单独端口模式

            即一个端口只服务于tcp代理的一组目的地址

            必须同时配置 `RouteId`

    - `HTTP1`/`HTTP2`/`HTTP3`

        http 代理， 可以同一端口同时服务于 `HTTP1`/`HTTP2`/`HTTP3`
        
        监听的端口将直接默认为复用， 通过 host 和 url 等等通过路由配置决定代理到哪一个目的地址

        不过目前 http3 由于 quic 底层实现限制， 无法在运行时动态切换证书，所以http3 暂时无法提供很灵活的复用，存在证书的限制

- `Address`

    监听的地址，支持多个

    格式为 `{ip}:{port}` ，比如 `127.0.0.1:80`

- `UseSni`

    为 true 时表明 tcp 代理 即 `"Protocols": ["TCP"]` 是否为 SNI 代理， udp 和 http 不可以配置

- `SniId`

    证书配置对应 id

    - tcp

        端口固定使用某一个证书对 tcp 采用 ssl 加密 以及 sni 路由设置

    - https

        端口固定使用某一个证书对 http 采用 ssl 加密， 
        
        HTTP1 和 HTTP2 支持通过SNI 匹配证书，所以复用端口场景无需配置

    - HTTP3

        必须为有效值， 因为目前 quic 底层实现限制， 无法在运行时动态切换证书，必须在建立时明确证书