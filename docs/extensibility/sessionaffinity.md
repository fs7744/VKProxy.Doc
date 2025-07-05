# 会话亲和性

会话亲和性是一种机制，用于将有因果关系的请求序列绑定到在多个目标之间均衡负载时处理第一个请求的目标。 在序列中的大多数请求处理相同数据以及处理请求的不同节点（目标）的数据访问成本不同的情况下，它很有用。 最常见的示例是暂时性缓存（例如内存中），其中第一个请求将数据从较慢的永久性存储提取到快速本地缓存中，而其他请求则只处理缓存的数据，从而增加吞吐量。

## 建立新的亲和性或解析现有的亲和性

请求到达并路由到启用了会话亲和性的群集后，代理会根据请求中的亲和性键的状态和有效性来自动决定是应建立新的亲和性，还是需要解析现有的亲和性，如下所示：

1. **请求不包含密钥**。 跳过解析，并与负载均衡器选择的目标建立新的亲和性

2. **请求中已找到关联密钥，并且有效**. 相关性机制会尝试查找与密钥匹配的第一个正常目标。

3. **关联键无效或找不到正常健康的关联目标**。 它被视为失败，并与负载均衡器选择的目标建立新的亲和性

如果为请求建立了新的相关性，则关联键会附加到响应，其中确切的键表示形式和位置取决于实现。 目前，有两个内置策略将密钥存储在cookie标头或自定义标头上。 将响应传递到客户端后，客户端负责将密钥附加到同一会话中的所有以下请求。 此外，当承载密钥的下一个请求到达代理时，它会解析现有相关性，但关联密钥不会再次附加到响应。 因此，只有第一个响应具有亲和性键。

有四种内置关联策略，这些策略在请求和响应上以不同的方式格式化和存储密钥。

- `HashCookie`

    策略使用 XxHash64 哈希为 cookie 值生成快速、紧凑、不明确的输出格式。

    请求的键将作为具有配置名称的 cookie 进行传递，并在关联序列中的第一个响应上使用 cookie 标头设置相同的 Set-Cookie。

- `ArrCookie`

    策略使用 SHA-256 哈希为cookie值生成模糊输出。

    请求的键将作为具有配置名称的 cookie 进行传递，并在关联序列中的第一个响应上使用 cookie 标头设置相同的 Set-Cookie。

- `Cookie`

    策略使用数据保护来加密密钥

    请求的键将作为具有配置名称的 cookie 进行传递，并在关联序列中的第一个响应上使用 cookie 标头设置相同的 Set-Cookie。

- `CustomHeader`

    策略使用数据保护来加密密钥，将密钥存储为加密标头。 它要求在具有已配置名称的自定义标头中传递关联键，并在关联序列中的第一个响应上设置相同的标头。

    由于 header 不具备 cookie 这般在浏览器等有内置附加在请求处理的逻辑，需要用户自行处理附加在后续的请求中。所以优先使用 cookie的方式

## 设置项

大家可以可以在cluster的`Metadata` 设置`SessionAffinity` 为 `HashCookie/ArrCookie/Cookie/CustomHeader` 其一来启用会话亲和性

- `CustomHeader`

    当 `SessionAffinityKey` 有设置时，其值将作为 header name，否则将默认采用 `x-sessionaffinity`

- `HashCookie/ArrCookie/Cookie`

    当 `SessionAffinityKey` 有设置时，其值将作为 cookie name，否则将默认采用 `SessionAffinity`

    而如下设置可以用于控制 cookie

    - `CookieDomain`

        获取或设置要与 Cookie 关联的域。
    
    - `CookieExpires`

        获取或设置 Cookie 的到期日期和时间。

    - `CookieExtensions`

        获取要追加到 Cookie 的其他值的集合。

    - `CookieHttpOnly`

        获取或设置一个值，该值指示客户端脚本是否无法访问 Cookie。 不设置则默认为 true

    - `CookieIsEssential`

        指示此 Cookie 是否对应用程序正常运行至关重要。 如果为 true，则可以绕过同意策略检查。 默认值为 false。

    - `CookieMaxAge`

        获取或设置 Cookie 的最大期限。

    - `CookiePath`

        获取或设置 Cookie 路径。 不设置则默认为"/"

    - `CookieSameSite`	

        获取或设置 Cookie 的 SameSite 属性的值。 默认值为 Unspecified

    - `CookieSecure`
    
        获取或设置一个值，该值指示是否使用安全套接字层（SSL）（即仅通过 HTTPS 传输 Cookie）。

配置示例：


``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "apidemo",
        "Metadata": {
          "MirrorCluster": "apidemoMirror"
        }
      }
    },
    "Clusters": {
      "apidemo": {
        "LoadBalancingPolicy": "RoundRobin",
        "Metadata": {
          "SessionAffinity": "Cookie",
          "CookieExpires": "00:00:13"
        },
        "Destinations": [
          {
            "Address": "https://xxx.lt"
          },
          {
            "Address": "https://xxx1.lt"
          },
          {
            "Address": "https://xxx2.lt"
          }
        ]
      }
    }
  }
}
```