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