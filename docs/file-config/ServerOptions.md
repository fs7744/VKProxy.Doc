# ServerOptions

# [appsettings.json](#tab/ServerOptions-json)

通过`appsettings.json`配置

``` json
{
  "ServerOptions": {
    "AddServerHeader": false
  },
}
```

# [code](#tab/ServerOptions-code)

通过`代码`配置

``` csharp
.ConfigureServices(i =>
{
    i.Configure<KestrelServerOptions>(o => o.AddServerHeader = false);
})
```

---

- `AddServerHeader`

    bool 类型, 默认值 false

    获取或设置是否 Server 应在每个响应中包含 标头。

- `DisableStringReuse`

    bool 类型, 默认值 false

    获取或设置一个值，该值控制是否将在请求之间重复使用具体化的字符串值;如果它们匹配，则为 ;如果字符串将始终重新分配

- `AllowAlternateSchemes`

    bool 类型, 默认值 false

    获取或设置一个值，该值控制如何 :scheme 验证 HTTP/2 和 HTTP/3 请求的 字段。

    如果 false HTTP/2 和 HTTP/3 请求的“：scheme”字段必须与传输 (完全匹配，例如，HTTPs 用于 TLS 连接，http 表示非 TLS) 。 如果 true 这样，HTTP/2 和 HTTP/3 请求的“：scheme”字段可以设置为备用值，这将由“HttpRequest.Scheme”反映。 根据 https://datatracker.ietf.org/doc/html/rfc3986/#section-3.1，方案必须仍然有效。 仅在使用受信任的代理时启用此功能。 这可用于从备用协议转换的代理等方案。 请参阅 https://datatracker.ietf.org/doc/html/rfc7540#section-8.1.2.3。 启用此功能的应用程序应在使用之前验证是否提供了预期方案。


- `AllowResponseHeaderCompression`

    bool 类型, 默认值 true

    获取或设置一个值，该值控制是否允许动态压缩响应标头。 有关 HPack 动态标头压缩的安全注意事项的详细信息，请访问 https://tools.ietf.org/html/rfc7541#section-7。

- `AllowSynchronousIO`

    bool 类型, 默认值 false

    获取或设置一个值，该值控制是否允许 和 使用 Request 同步 IO Response

- `Limits`

    提供对请求限制选项的访问权限。

    - `KeepAliveTimeout`

        TimeSpan 类型, 默认值为 130 秒。

        获取或设置保持活动状态超时。 

    - `RequestHeadersTimeout`

        TimeSpan 类型, 默认值为 30 秒。

        获取或设置服务器在接收请求标头时花费的最长时间。

    - `MaxConcurrentConnections`

        int 类型,  默认为“null”。

        获取或设置打开连接的最大数目。 如果设置为 null，则连接数不受限制。

    - `MaxConcurrentUpgradedConnections`

        int 类型,  默认为“null”。

        获取或设置打开的已升级连接的最大数目。 如果设置为 null，则升级的连接数不受限制。 升级的连接是已从 HTTP 切换到另一个协议（如 WebSocket）的连接。

    - `MaxRequestBufferSize`

        long 类型, 默认为 1，048，576 字节 (1 MB) 。

        获取或设置请求缓冲区的最大大小。 

    - `MaxRequestBodySize`

        long 类型, 默认为 30，000，000 字节，大约为 28.6MB。

        获取或设置任何请求正文允许的最大大小（以字节为单位）。 如果设置为 null，则最大请求正文大小不受限制。 此限制对始终不受限制的升级连接没有影响。 这可以通过 按请求 IHttpMaxRequestBodySizeFeature重写。 

    - `MaxRequestLineSize`

        long 类型, 默认为 8，192 字节 (8 KB) 。

        获取或设置 HTTP 请求行允许的最大大小。 

    - `MaxRequestHeadersTotalSize`

        long 类型, 默认为 32，768 字节 (32 KB) 。

        获取或设置 HTTP 请求标头允许的最大大小。 

    - `MaxRequestHeaderCount`

        int 类型, 默认为 100。

        获取或设置每个 HTTP 请求允许的最大标头数。 

    - `MinRequestBodyDataRate`

        MinDataRate 类型, 默认为 240 字节/秒，宽限期为 5 秒。

        获取或设置请求正文最小数据速率（以字节/秒为单位）。 将此属性设置为 null 表示不应强制实施最低数据速率。 此限制对始终不受限制的升级连接没有影响。 这可以通过 按请求 IHttpMinRequestBodyDataRateFeature重写。 

    - `MinResponseDataRate`

        MinDataRate 类型, 默认为 240 字节/秒，宽限期为 5 秒。

        获取或设置响应最小数据速率（以字节/秒为单位）。 将此属性设置为 null 表示不应强制实施最低数据速率。 此限制对始终不受限制的升级连接没有影响。 这可以通过 按请求 IHttpMinResponseDataRateFeature重写。

    - `MaxResponseBufferSize`

        long 类型, 默认为 65，536 字节 (64 KB) 。

        获取或设置在写入调用开始阻止或返回在缓冲区大小低于配置限制之前未完成的任务的响应缓冲区的最大大小。 

    - `Http2`

        限制仅适用于 HTTP/2 连接。

        - `HeaderTableSize`

            int 类型, 默认为 4096 个八进制 (4 KiB) 。

            限制服务器上的 HPACK 编码器和解码器可以使用的标头压缩表的大小（以八进制为单位）。

            值必须大于或等于 0

        - `InitialConnectionWindowSize`

            long 类型, 值必须大于或等于 64 KiB 且小于 2 GiB，默认值为 1 MiB。

            指示服务器一次愿意接收和缓冲的请求正文数据量（以字节为单位），这些数据聚合到每个连接的所有请求 (流) 。 

        - `InitialStreamWindowSize`

            long 类型, 值必须大于或等于 64 KiB 且小于 2 GiB，默认值为 768 KiB。

            指示服务器一次愿意接收和缓冲每个流的请求正文数据量（以字节为单位）。 请注意，连接也受 InitialConnectionWindowSize限制。 流窗口和连接窗口中都必须有空间，以便客户端上传请求正文数据。

        - `KeepAlivePingDelay`

            TimeSpan 类型, 延迟值必须大于或等于 1 秒。 MaxValue设置为 可禁用保持活动状态 ping。 默认为 MaxValue。

            获取或设置保持活动状态 ping 延迟。 如果服务器在此时间段内未在连接上收到任何帧，则服务器将向客户端发送保持活动 ping。 此属性与 KeepAlivePingTimeout 一起使用以关闭断开的连接。

        - `KeepAlivePingTimeout`

            TimeSpan 类型, 超时必须大于或等于 1 秒。 设置为 MaxValue 以禁用保持活动状态 ping 超时。 默认为 20 秒。

            获取或设置保持活动状态 ping 超时。 当处于非活动状态的时间段超过配置 KeepAlivePingDelay 的值时，将发送保持活动状态 ping。 如果在超时时间内未收到任何帧，服务器将关闭连接。

        - `MaxFrameSize`

            long 类型, 值必须介于 2^14 和 2^24 之间，默认为 2^14 个八进制 (16 KiB) 。

            指示允许接收的最大帧有效负载的大小（以八位字节为单位）。 大小必须介于 2^14 和 2^24-1 之间。

        - `MaxRequestHeaderFieldSize`

            long 类型, 值必须大于 0，默认值为 2^14 个八进制 (16 KiB) 。

            指示请求标头字段序列允许的最大大小的大小（以八位字节为单位）。 此限制适用于其压缩和未压缩表示形式的名称和值序列。

        - `MaxStreamsPerConnection`

            int 类型, 值必须大于 0，默认为 100 个流。

            制每个 HTTP/2 连接的并发请求流的数量。 过多的流将被拒绝。

    - `Http3`

        限制仅适用于 HTTP/3 连接。

        - `MaxRequestHeaderFieldSize`

            long 类型, 值必须大于 0，默认值为 2^14 (16，384) 。

            指示请求标头字段序列允许的最大大小的大小。 此限制适用于其压缩和未压缩表示形式的名称和值序列。