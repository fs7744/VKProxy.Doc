# Routes

路由在不同场景有不同侧重

- L4 

    由于真实路由实际是端口区别或者同SNI 处理的，此场景其实不具备路由的功能

    但是考虑到大家在L4场景有更多的扩展需求，比如扩展协议 实现 socks5 等等，而外的配置参数就是很必要了，所以依然沿用Route，用户可以在 `Metadata` 任意添加自己所需参数（虽然对配置稍显复杂）

- L7

    其实就是 http 路由匹配， 此场景兼具路由和扩展需求

如下则一个最简单的示例

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

``` json
{
  "ReverseProxy": {
    "Routes": {
        // Routes id : tcpTestRoute
      "tcpTestRoute": {
        "ClusterId": "tcpTestCluster",  // tcp 目的地址 id
        "Timeout": "00:00:11"  // 超时时间 11 秒， 
      }
    }
  }
}
```

---

路由同样每一个id必须唯一，不能重复，因为运行时会根据配置变动动态调整监听，不必重启实例

路由 有以下配置项

- `Order`

    int 类型

    优先级，数字越小优先级越高， 您可以使用优先级在比如 http 路由冲突场景确定顺序，避免冲突

    默认Http路由匹配优先顺序为 1）Host (如冲突则会按照 Order依次匹配), 2）Path (如冲突则会按照 Order依次匹配)， 3）路由其他匹配

- `Timeout`

    TimeSpan 类型，例如配置文件可配`"00:05:00"`

    代理处理超时时间

- `ClusterId`

    string 类型

    负载均衡配置对应id

- `UdpResponses`

    int 类型

    接受服务端最多返回多少个 udp 包

- `Match`

    http 路由匹配规则，其可以配置如下参数

    - `Hosts`

        string[] 类型

        域名匹配列表， 可以配置以下格式

        - 精确匹配

            完全按照host一模一样对比，例如`Hosts:["a.com"]` 只能匹配 a.com ，不能匹配 aa.com

        - 后缀匹配

            后缀一致的匹配方式，例如`Hosts:["*a.com"]` 能匹配 aa.com ，不能匹配 ab.com

        - 任意

            `Hosts:["*"]` 这样任意域名都会匹配上，无论 aa.com 还是 bb.org

    - `Paths`

        string[] 类型

        Path (url 除去域名 querystring 的部分) 匹配列表， 可以配置以下格式

        - 精确匹配

            完全按照host一模一样对比，例如`Paths:["/a"]` 只能匹配 /a ，不能匹配 /ab

        - 前缀匹配

            前缀一致的匹配方式，例如`Paths:["/a*"]` 能匹配 /ab ，不能匹配 /bb

        - 任意

            `Paths:["*"]` 这样任意url都会匹配上，无论 /a 还是 /b 还是 /

    - `Methods`

        string[] 类型

        允许 Method 的范围列表，即白名单， 如 `Methods:["GET"]` 只允许 GET 请求， 任意 POST 等其他请求都会 404

    - `Statement`

        string 类型

        提供 类似 sql 中 where 的简单表达式写法, 以满足大家在路由匹配时更为复杂的匹配场景，当然有一些限制 ， 
        
        比如`"Statement": "Method = 'GET'"` 就是 http method 匹配 GET ，
        
        其实现为 解析 表达式字符串然后生成 `Func<HttpContext, bool>`, 所以虽然性能不是最优，但也不算太差
        
        请参见 [如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement) 具体说明

- `Transforms`

    Dictionary<string, string>[] 类型

    可以配置修改 http header 等等场景

    请参见 [如何为HTTP配置请求和响应转换](/VKProxy.Doc/docs/transforms) 具体说明

- `Metadata`

    Dictionary<string, string> 类型

    可以配置用于自定义扩展场景时提供给配置用户的参数

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

        根据 header 提供提供不同的速率控制计数  （请求 header 没有值时会默认采用 `RemoteIpAddress`）

        比如 cdn 会返回真实客户端ip， （如 X-forwarded-For）， 你可以根据此header限制每一个ip 每10秒只能3个请求

    - `Cookie`

        根据 Cookie 提供提供不同的速率控制计数  （请求 Cookie 没有值时会默认采用 `RemoteIpAddress`）

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