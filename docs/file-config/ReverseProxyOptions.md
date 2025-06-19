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

- `Limit`

    速率限制是限制可以访问的资源量的概念。 例如，你可能知道应用访问的数据库每分钟可以安全地处理 1,000 个请求，但它可能处理不了更多。 可以在应用中放置一个速率限制器，每分钟只允许 1,000 个请求，并在它们可以访问数据库之前拒绝更多请求。 因此，限制数据库速率并允许应用处理安全数量的请求。 这是分布式系统中的一种常见模式，其中你可能有多个应用实例正在运行，并且你希望确保它们不会同时尝试访问数据库。 有多个不同的速率限制算法来控制请求流。

    可全局设置速率限制，但是当路由有单独速率限制时优先使用路由级别

    - `By`

        可以划分速率限制的粒度， 目前支持两种 `Total`/ `Key`
        - `Total`

            整个路由采用同一份速率控制，无论不同ip还是不同用户

        - `Key`

            根据`Header`和`Cookie`设置提供不同的速率控制计数

    - `Header`

        根据 header 提供提供不同的速率控制计数

        比如 cdn 会返回真实客户端ip， （如 X-forwarded-For）， 你可以根据此header限制每一个ip 每10秒只能3个请求

    - `Cookie`

        根据 Cookie 提供提供不同的速率控制计数

        比如 Cookie中有唯一用户id， （如 user-id）， 你可以根据此Cookie限制每一个用户 每10秒只能3个请求

    - `Policy`

        目前支持四种策略

        - `Concurrency`

            并发限制器会限制并发请求数。 每添加一个请求，在并发限制中减去 1。 一个请求完成时，在限制中增加 1。 其他请求限制器限制的是指定时间段的请求总数，而与它们不同，并发限制器仅限制并发请求数，不对一段时间内的请求数设置上限。

            此策略有效参数有 `PermitLimit` / `QueueLimit`

        - `FixedWindow`

            使用固定的时间窗口来限制请求。 当时间窗口过期时，会启动一个新的时间窗口，并重置请求限制。

            此策略有效参数有 `PermitLimit` / `QueueLimit`/ `Window`

        - `SlidingWindow`

            滑动窗口算法：

            - 与固定窗口限制器类似，但为每个窗口添加了段。 窗口在每个段间隔滑动一段。 段间隔的计算方式是：(窗口时间)/(每个窗口的段数)。
            - 将窗口的请求数限制为 permitLimit 个请求。
            - 每个时间窗口划分为一个窗口 n 个段。
            - 从倒退一个窗口的过期时间段（当前段之前的 n 个段）获取的请求会添加到当前的段。 我们将倒退一个窗口最近过期时间段称为“过期的段”。

            此策略有效参数有 `PermitLimit` / `QueueLimit`/ `Window` / `SegmentsPerWindow`

        - `TokenBucket`

            令牌桶限流器与滑动窗口限制器类似，但是它并不重新添加来自过期段的请求，而是在每个补充周期内一次性补充固定数量的令牌。 每个段添加的令牌数不能使可用令牌数超过令牌桶限制。 下表显示了一个令牌桶限制器，其中令牌数限制为 100 个，补充期为 10 秒。

            此策略有效参数有 `PermitLimit` / `QueueLimit`/ `Window` / `TokensPerPeriod`

    - `PermitLimit`

        在同时租用的最大许可证数。

    - `QueueLimit`

        当已达限制时可同时排队的最大许可数。当为 0 时，表示禁止队列

    - `SegmentsPerWindow`

        指定窗口划分到的最大段数。

    - `Window`

        指定补货之间的最短期限。TimeSpan 格式
    
    - `TokensPerPeriod`

        指定用于还原每次补充的最大令牌数