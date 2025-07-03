# 什么是速率限制？

速率限制是限制可以访问的资源量的概念。 例如，你可能知道应用访问的数据库每分钟可以安全地处理 1,000 个请求，但它可能处理不了更多。 可以在应用中放置一个速率限制器，每分钟只允许 1,000 个请求，并在它们可以访问数据库之前拒绝更多请求。 因此，限制数据库速率并允许应用处理安全数量的请求。 这是分布式系统中的一种常见模式，其中你可能有多个应用实例正在运行，并且你希望确保它们不会同时尝试访问数据库。 有多个不同的速率限制算法来控制请求流。

在 .NET 中已经提供相关实现，可使用 System.Threading.RateLimiting NuGet 包。

VKProxy 也是基于其提供速率限制功能，不过相关实现都只是单实例内部计数，没有分布式的实现，毕竟分布式计数可能导致复杂度和性能下降。

# 为何使用速率限制
速率限制可用于管理向应用发出的传入请求流。 实现速率限制的关键原因：

- **防止滥用**：速率限制通过限制用户或客户端在给定时间段内发出的请求数来帮助保护应用免受滥用。 这对于公共 API 尤其重要。
- **确保公平使用**：通过设置限制，所有用户都可以公平地访问资源，防止用户垄断系统。
- **保护资源**：速率限制通过控制可处理的请求数来帮助防止服务器重载，从而防止后端资源过载。
- **增强安全性**：它可以通过限制处理请求的速度来缓解拒绝服务（DoS）攻击的风险，从而使攻击者更难淹没系统。
- **提高性能**：通过控制传入请求的速度，可以维护应用的最佳性能和响应能力，确保更好的用户体验。
- **成本管理**：对于基于使用情况产生成本的服务，速率限制可以通过控制处理的请求量来帮助管理和预测费用。

速率限制已经是API Gateway标配功能，所以VKProxy也得有

# 防止 DDoS 攻击
虽然速率限制通过限制处理请求的速率来帮助缓解拒绝服务（DoS）攻击的风险，但它不是分布式拒绝服务（DDoS）攻击的综合解决方案。 DDoS 攻击涉及多个系统，使应用遭受大量请求，因此难以单独处理速率限制。

对于可靠的 DDoS 保护，请考虑使用商业 DDoS 保护服务。 这些服务提供高级功能，例如：

- **流量分析**：持续监视和分析传入流量，实时检测和缓解 DDoS 攻击。
- **可伸缩性**：通过跨多个服务器和数据中心分布流量来处理大规模攻击的能力。
- **自动缓解**：自动响应机制，无需手动干预即可快速阻止恶意流量。
- **全球网络**：一个全局服务器网络，用于吸收和缓解离源更近的攻击。
- **不断更新**：商业服务持续跟踪和更新其保护机制，以适应新的和不断演变的威胁。

使用云托管服务时，DDoS 防护通常作为托管解决方案的一部分提供，例如 Azure Web 应用程序防火墙、 AWS 防护 或 Google Cloud Armor。 专用保护可用作 Web 应用程序防火墙（WAF）或 CDN 解决方案的一部分，例如 Cloudflare 或 Akamai Kona Site Defender

结合速率限制实施商业 DDoS 保护服务可以提供全面的防御策略，确保应用的稳定性、安全性和性能。

# 速率限制器算法

目前VKProxy提供四种速率限制

- `Concurrency` 并发
- `FixedWindow` 固定窗口
- `SlidingWindow` 滑动窗口
- `TokenBucket` 令牌桶

## `Concurrency` 并发

并发限制器会限制并发请求数。 每添加一个请求，在并发限制中减去 1。 一个请求完成时，在限制中增加 1。 其他请求限制器限制的是指定时间段的请求总数，而与它们不同，并发限制器仅限制并发请求数，不对一段时间内的请求数设置上限。

配置举例

# [appsettings.json](#tab/Concurrency-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        "Limit": {
            "Policy": "Concurrency",
            "By": "Total",      // 整个路由采用同一份速率控制，无论不同ip还是不同用户
            "PermitLimit": 10,  // 在同时租用的最大许可证数。
            "QueueLimit": 0    // 当已达限制时可同时排队的最大许可数。当为 0 时，表示禁止队列
        },
      }
    }
  }
}
```


# [ui](#tab/Concurrency-ui)

![limit-concurrency.jpg](/VKProxy.Doc/images/limit-concurrency.jpg)

---

## `FixedWindow` 固定窗口

使用固定的时间窗口来限制请求。 当时间窗口过期时，会启动一个新的时间窗口，并重置请求限制。

配置举例

# [appsettings.json](#tab/FixedWindow-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        "Limit": {
            "Policy": "FixedWindow",
            "By": "Key",      // 根据 header 提供提供不同的速率控制计数  
            "Header": "X-forwarded-For",  // 根据cdn 的客户端真实ip区分, 不同cdn 会有不同ip header , 没有值时会默认采用 `RemoteIpAddress`
            "PermitLimit": 10,  // 在同时租用的最大许可证数。
            "QueueLimit": 2,    // 当已达限制时可同时排队的最大许可数。当为 0 时，表示禁止队列
            "Window": "00:00:10"         // 指定补货之间的最短期限
        },
      }
    }
  }
}
```


# [ui](#tab/FixedWindow-ui)

![limit-fixedwindow.jpg](/VKProxy.Doc/images/limit-fixedwindow.jpg)

---

## `SlidingWindow` 滑动窗口

滑动窗口算法：

- 与固定窗口限制器类似，但为每个窗口添加了段。 窗口在每个段间隔滑动一段。 段间隔的计算方式是：(窗口时间)/(每个窗口的段数)。
- 将窗口的请求数限制为 permitLimit 个请求。
- 每个时间窗口划分为一个窗口 n 个段。
- 从倒退一个窗口的过期时间段（当前段之前的 n 个段）获取的请求会添加到当前的段。 我们将倒退一个窗口最近过期时间段称为“过期的段”。

请考虑下表，其中显示了一个滑动窗口限制器，该限制器的窗口为 30 秒、每个窗口有三个段，且请求数限制为 100 个：

- 第一行和第一列显示时间段。
- 第二行显示剩余的可用请求数。 其余请求数的计算方式为可用请求数减去处理的请求数和回收的请求数。
- 每次的请求数沿着蓝色对角线移动。
- 从时间 30 开始，从过期时间段获得的请求会再次添加到请求数限制中，如红色线条所示。

![rate.png](https://learn.microsoft.com/zh-cn/aspnet/core/performance/rate-limit/_static/rate.png?view=aspnetcore-9.0)

配置举例

# [appsettings.json](#tab/SlidingWindow-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        "Limit": {
            "Policy": "SlidingWindow",
            "By": "Key",      // 根据 Cookie 提供提供不同的速率控制计数  
            "Cookie": "sessionid",  // 根据Cookie的sessionid 区分, 没有值时会默认采用 `RemoteIpAddress`
            "PermitLimit": 10,  // 在同时租用的最大许可证数。
            "QueueLimit": 2,    // 当已达限制时可同时排队的最大许可数。当为 0 时，表示禁止队列
            "Window": "00:00:10",         // 指定补货之间的最短期限
            "SegmentsPerWindow": 2   // 指定窗口划分到的最大段数。
        },
      }
    }
  }
}
```


# [ui](#tab/SlidingWindow-ui)

![limit-slidingwindow.jpg](/VKProxy.Doc/images/limit-slidingwindow.jpg)

---

## `TokenBucket` 滑动窗口

令牌桶限流器与滑动窗口限制器类似，但是它并不重新添加来自过期段的请求，而是在每个补充周期内一次性补充固定数量的令牌。 每个段添加的令牌数不能使可用令牌数超过令牌桶限制。 下表显示了一个令牌桶限制器，其中令牌数限制为 100 个，补充期为 10 秒。

配置举例

# [appsettings.json](#tab/TokenBucket-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        "Limit": {
            "Policy": "TokenBucket",
            "By": "Total",      // 整个路由采用同一份速率控制，无论不同ip还是不同用户
            "PermitLimit": 10,  // 存储桶中随时可以包含的最大令牌数。
            "QueueLimit": 2,    // 当已达限制时可同时排队的最大许可数。当为 0 时，表示禁止队列
            "Window": "00:00:10",         // 指定补货之间的最短期限
            "TokensPerPeriod": 2   // 指定用于还原每次补充的最大令牌数
        },
      }
    }
  }
}
```


# [ui](#tab/TokenBucket-ui)

![limit-tokenbucket.jpg](/VKProxy.Doc/images/limit-tokenbucket.jpg)

---

## `RedisConcurrency` 并发

Redis版本并发限制器会限制并发请求数，主要表明大家可以利用扩展实现自己的并发机制。 每添加一个请求，在并发限制中减去 1。 一个请求完成时，在限制中增加 1。 其他请求限制器限制的是指定时间段的请求总数，而与它们不同，并发限制器仅限制并发请求数，不对一段时间内的请求数设置上限。

配置举例

# [appsettings.json](#tab/Concurrency-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        "Limit": {
            "Policy": "Concurrency",
            "By": "RedisConcurrency",      
            "PermitLimit": 10,  // 在同时租用的最大许可证数。
        },
      }
    }
  }
}
```

---