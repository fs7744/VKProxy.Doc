# QuicTransportOptions

[Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel) 内部对于Quic处理的配置 

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
    i.Configure<QuicTransportOptions>(o => o.Backlog = 512);
})
```

---

- `Backlog`

    int 类型, 默认值 512

    挂起的连接队列的最大长度。

- `DefaultCloseErrorCode`

    long 类型

    释放打开的连接时使用的错误代码。

- `DefaultStreamErrorCode`

    long 类型

    当流需要在内部中止流的读取或写入端时使用的错误代码。

- `MaxBidirectionalStreamCount`

    int 类型, 默认值 100

    每个连接的最大并发双向流数。

     **RequiresPreviewFeatures** Preview 功能，不支持json配置

- `MaxReadBufferSize`

    long 类型, 默认值  1024 * 1024

    最大读取大小。

     **RequiresPreviewFeatures** Preview 功能，不支持json配置

- `MaxUnidirectionalStreamCount`

    int 类型, 默认值  10

    每个连接的最大并发入站单向流数。

     **RequiresPreviewFeatures** Preview 功能，不支持json配置

- `MaxWriteBufferSize`

    long 类型, 默认值  64 * 1024

    最大写入大小。

     **RequiresPreviewFeatures** Preview 功能，不支持json配置