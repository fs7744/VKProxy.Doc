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