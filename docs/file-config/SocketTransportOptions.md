# UdpSocketTransportOptions

[Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel) 内部对于tcp处理的配置 

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

``` json
{
  "ServerOptions": {
    "Socket": {
        "Backlog": 512
    }
  },
}
```

# [code](#tab/code)

通过`代码`配置

``` csharp
.ConfigureServices(i =>
{
    i.Configure<SocketTransportOptions>(o => o.Backlog = 512);
})
```

---

- `Backlog`

    int 类型, 默认值 512

    挂起的连接队列的最大长度。

- `IOQueueCount`

    int 类型, 默认值  System.Environment.ProcessorCount

    用于处理请求的 I/O 队列数。 设置为 0 可直接将 I/O 计划到 ThreadPool。

- `MaxReadBufferSize`

    long 类型, 默认值  1 MiB.

    获取或设置传输将缓冲的最大未用量传入字节数。

    值 null 或 0 完全禁用反压，允许无限缓冲。 鉴于不受信任的客户端，无限服务器缓冲是一种安全风险。

- `MaxWriteBufferSize`

    long 类型, 默认值 64 KiB

    获取或设置传输在应用写回压之前将缓冲的最大传出字节数。

    值 null 或 0 完全禁用反压，允许无限缓冲。 鉴于不受信任的客户端，无限服务器缓冲是一种安全风险。

- `NoDelay`

    int 类型, 默认值  true

    设置为 false 可对所有连接启用 Nagle 算法。

- `UnsafePreferInlineScheduling`

    bool 类型, 默认值  false

    内联应用程序和传输延续，而不是调度到线程池。

- `WaitForDataBeforeAllocatingBuffer`

    bool 类型, 默认值  true

    等到有数据可用于分配缓冲区。 将此设置为 false 可能会增加吞吐量，但代价是内存使用量增加。