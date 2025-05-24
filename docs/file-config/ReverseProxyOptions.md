# ReverseProxyOptions

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

``` json
{
  "ReverseProxy": {
    "DefaultProxyTimeout": "00:05:00"
  },
}
```

# [code](#tab/code)

通过`代码`配置

``` csharp
.ConfigureServices(i =>
{
    i.Configure<ReverseProxyOptions>(o => o.DefaultProxyTimeout = TimeSpan.FromSeconds(300));
})
```

---

- `DefaultProxyTimeout`

    TimeSpan 类型，例如配置文件可配`"00:05:00"`，或代码中可配`TimeSpan.FromSeconds(300)`， 默认值 300 秒

    默认代理处理超时时间，只有当路由没有配置超时时间时才会使用

- `ConnectionTimeout`

    TimeSpan 类型，例如配置文件可配`"00:00:03"`，或代码中可配`TimeSpan.FromSeconds(3)`， 默认值 3 秒

    建立连接超时时间， 比如 tcp 的 connect

- `RouteCahceSize`

    int 类型，默认值 1024

    比如 http url 前缀匹配或者 sni host后缀匹配等路由匹配场景中，复用相同匹配值进而提升性能的缓存大小

- `RouteComparison`

    StringComparison 类型，默认值

    路由缓存或者比如 http url 匹配或者 sni host 匹配是否区分大小写

    具体值如下

    ``` csharp
    public enum StringComparison
    {
        //
        // Summary:
        //     Compare strings using culture-sensitive sort rules and the current culture.
        CurrentCulture = 0,
        //
        // Summary:
        //     Compare strings using culture-sensitive sort rules, the current culture, and
        //     ignoring the case of the strings being compared.
        CurrentCultureIgnoreCase = 1,
        //
        // Summary:
        //     Compare strings using culture-sensitive sort rules and the invariant culture.
        InvariantCulture = 2,
        //
        // Summary:
        //     Compare strings using culture-sensitive sort rules, the invariant culture, and
        //     ignoring the case of the strings being compared.
        InvariantCultureIgnoreCase = 3,
        //
        // Summary:
        //     Compare strings using ordinal (binary) sort rules.
        Ordinal = 4,
        //
        // Summary:
        //     Compare strings using ordinal (binary) sort rules and ignoring the case of the
        //     strings being compared.
        OrdinalIgnoreCase = 5
    }
    ```

- `DnsRefreshPeriod`

    TimeSpan 类型，例如配置文件可配`"00:05:03"`，或代码中可配`TimeSpan.FromMinutes(5)`， 默认值 5 分钟

    DNS 目标解析请求刷新已解析名称之间的时间间隔

- `DnsAddressFamily`

    `InterNetwork` 或 `InterNetworkV6` 默认值 null

    将解析限制为 IPv4 或 IPv6 地址。 默认值 null 指示解析程序不限制结果的地址系列，并使用接受所有返回的地址。
