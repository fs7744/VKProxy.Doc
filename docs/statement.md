# 如何为HTTP配置路由复杂匹配

http 路由匹配规则可以配置 `Hosts` / `Paths` / `Methods`

虽然可以满足大部分简单场景，但是只能过滤交集以及header等等其他场景无法过滤，

而为了大家能更加语义化描述如何过滤，并且最好降低大家学习成本，所以尝试实现了一套简单的类似 sql 中 where 部分表达式解析实现

其表达式定义为 `([Http字段] [比较操作符] [比较值]) [可选： [AND/OR ([Http字段] [比较操作符] [比较值])] ...]`

比如大家就可以描述为 `Method = 'GET'` 或者 `Method = 'GET' OR Method = 'POST'`

其实现为 解析 表达式字符串然后生成 `Func<HttpContext, bool>`, 所以虽然性能不是最优，但理论上也不算太差

> [!WARNING]
> 为了避免大家错误描述表达式造成 `1 = 1` 等表达式造成路由匹配异常，所以限制必须`[Http字段] [比较操作符] [比较值]`， 不能`[比较值] [比较操作符] [比较值]`

### 比较值

支持以下三种格式

- bool

    如 `true` or `false`
- number

    如 `12323` or `1.324` or `-44.4`
- string

    如 `'sdsdfa'` or `'sds\'dfa'` or `"dsdsdsd"` or `"fs\"dsf"`

### Http字段

由于是用于路由匹配，所以只支持 [`HttpRequest`](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.aspnetcore.http.httprequest?view=aspnetcore-9.0) 部分字段

支持如下：

- `Path`

    获取或设置标识所请求资源的请求路径部分。

    如果 PathBase 包含完整路径，则该值可以是 Empty ;对于“OPTIONS *”请求，该值可以是 。 除“%2F”外，服务器将完全解码路径，该路径将解码为“/”并更改路径段的含义。 “%2F”只能在将路径拆分为段后替换。
- `PathBase`

    获取或设置请求的基路径。 路径基不应以尾部斜杠结尾。
- `Method`

    获取或设置 HTTP 方法。
- `Scheme`

    获取或设置 HTTP 请求方案。
- `IsHttps`

    如果 RequestScheme 为 https，则返回 true。
- `Protocol`

    获取或设置请求协议 (例如 HTTP/1.1) 。
- `ContentType`

    获取或设置 Content-Type 标头。
- `ContentLength`

    获取或设置 Content-Length 标头。
- `Host`

    获取或设置 Host 标头。 可以包含端口。
- `QueryString`

    获取或设置用于在 Request.Query 中创建查询集合的原始查询字符串。
- `HasFormContentType`

    检查表单类型的 Content-Type 标头。

如下为动态集合字段，需要再指定Key, 格式为 `[Http字段]([Key])`, 如获取User-Agent则为`Header('User-Agent')`

- `Header`

    获取请求标头。
- `Query`

    获取从 Request.QueryString 分析的查询值集合。
- `Cookie`

    获取此请求的 Cookie 集合。
- `Form`

    获取或设置窗体形式的请求正文。

### 比较操作符

- `=`  

    Equal

    如 ` ContentType = 'sky'`
- `<=`  

    LessThan or Equal 

    如 ` ContentLength <= 30`
- `<`  

    LessThan 

    如 ` ContentLength < 30`
- `>=`  

    GreaterThan or Equal

    如 ` ContentLength >= 30`

- `>`  

    GreaterThan 

    如 ` ContentLength > 30`
- `!=`  

    Not Equal 

    如 ` ContentLength != 30`
- `~= '正则表达式'`  

    正则匹配

    如 `Path ~= '[/]test.*'`
- `in ()`  

    in array (bool/number/string)

    如 `in (1,2,3)` or `in ('sdsdfa','sdfa')` or `in (true,false)`
- `not`

    取反

    如 ` not( ContentLength <= 30 )`
- `and`

    如 `  ContentLength <= 30 and ContentLength > 60`
- `or`

    如 ` ContentLength <= 30 or ContentLength > 60`
- `()`

    如 ` (ContentLength <= 30 or ContentLength > 60) and ContentType = 'killer'`