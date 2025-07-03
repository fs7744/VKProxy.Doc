# http响应缓存

响应缓存可减少客户端或代理对 Web 服务器发出的请求数。 响应缓存还减少了 Web 服务器为生成响应而执行的工作量。 响应缓存在标头中设置。

客户端和中间代理应遵循 [ RFC 9111：HTTP ](https://www.rfc-editor.org/rfc/rfc9111) 缓存下缓存响应的标头。

目前暂时内置只内存缓存，不过可以通过扩展接口扩展

## 缓存条件

- 请求条件

    - 默认条件
        - 请求方法必须是 `GET` 或 `HEAD`。
        - 不能出现 `Authorization` 标头。
        - 不能存在 `Cache-Control: no-cache` 或 `Pragma: no-cache`
        - 不能存在 `Cache-Control: no-store`

    - 自定义条件

        可以通过在`Metadata`中设置 `CacheWhen` 自定义条件(一旦设置，默认条件将不起作用)， 格式参见 [如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)

        不过依然不能存在 `Cache-Control: no-store`，避免缓存错误时无法处理

- 响应条件

    - 缓存协商条件
        - 请求必须生成带有 200 (OK) 状态代码的服务器响应。
        - 不能出现 Set-Cookie 标头。 
        - Cache-Control 标头参数必须是有效的，并且必须将响应标记为 public 而不是 private。
        - 不能存在 `Cache-Control: no-store`
        - 不能存在 `Cache-Control: no-cache`
        - Vary 标头参数必须有效且不等于 *。
        - Content-Length 标头值（若已设置）必须与响应正文的大小匹配。
        - 根据 Expires 标头与 max-age 和 s-maxage 缓存指令所指定，响应不能过时。
        - 响应缓冲必须成功。 响应的大小必须小于配置的或默认的 SizeLimit。 响应的正文大小必须小于配置的或默认的 MaximumBodySize。
        - 响应必须可根据 [RFC 9111：HTTP](https://www.rfc-editor.org/rfc/rfc9111) 缓存进行缓存。 例如，no-store 指令不能出现在请求头或响应头字段中。 有关详细信息，请参阅 RFC 9111：HTTP 缓存（第 3 节“在缓存中存储响应”）。

    - 强制缓存

        可以通过在`Metadata`中设置 `ForceCache` 为 `true` 忽略响应缓存条件标准，不过程序如果遵从响应缓存协商标准还是使用标准最好，以免缓存错误

        强制缓存条件下，只有以下限制：

        - 请求必须生成带有 200 (OK) 状态代码的服务器响应。
        - 不能出现 Set-Cookie 标头。

## 缓存设置

大家可以在`Metadata`中设置缓存， 具体设置项如下

- `Cache`

    缓存方式， 内置缓存有 (还可以通过扩展接口扩展，然后大家在此设置)
    
    - `Memory`

        基于 MemoryCache 实现

    - `Disk`

        缓存会实际存于磁盘物理文件

    - `Redis`

        启用 Redis 情况还可以缓存导redis中

- `CacheWhen`

    设置自定义条件(一旦设置，默认条件将不起作用,不设置则使用默认条件)， 格式参见 [如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)

- `CacheKey`

    通过内置“简略版模板引擎”，大家可以灵活设置缓存key，如 `"CacheKey": "{Method}_{Path}"` 效果等同于 `$"{RouteKey}#{Method}_{Path}"`, 值结果可为 `Route1#GET_/FAQ`

    具体设置内容后文详细说明，当然如模板效果不够，大家还可以通过扩展替换引入真正更为强大的模板引擎

- `CacheMaximumBodySize`

    可以设置允许缓存Body的最大大小，超过则不会缓存，默认 64 * 1024 * 1024 ,即 64MB

- `ForceCache`

    设置 `ForceCache` 为 `true` 忽略响应缓存条件标准，不过程序如果遵从响应缓存协商标准还是使用标准最好，以免缓存错误

    强制缓存条件下，只有以下限制：

    - 请求必须生成带有 200 (OK) 状态代码的服务器响应。
    - 不能出现 Set-Cookie 标头。

- `CacheTime`

    缓存时间，TimeSpan 格式，不设置采用标头中缓存协商结果，否则使用设置值

## `CacheKey`内置“简略版模板引擎”

为了方便大家使用，内置实现了一套简单的数据源为 `HttpContext`的“简略版模板引擎”，

格式为 `{数据}`，只要被`{}`包裹的数据都会在运行时替换为当前实时数据结果， (如需原样`{}`，则可通过双重转义， 如 `{{123}}{Path}` 结果为 `{123}/FAQ`)

具体数据可采用列表如下

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

### 可通过扩展替换模板引擎

如简单的不足以满足大家所需，大家可以通过扩展替换模板引擎，

只需实现如下接口

``` csharp
public interface ITemplateStatementFactory
{
    public Func<HttpContext, string> Convert(string template);
}
```

然后通过 di 替换

``` csharp
services.AddSingleton<ITemplateStatementFactory, XXXTemplateStatementFactory>();
```

## 可通过扩展替换缓存存储

如内存缓存不足以满足大家所需，大家可以通过扩展替换

只需实现如下接口

``` csharp
public interface IResponseCache
{
    string Name { get; }

    ValueTask<CachedResponse?> GetAsync(string key, CancellationToken cancellationToken);

    ValueTask SetAsync(string key, CachedResponse entry, TimeSpan validFor, CancellationToken cancellationToken);
}
```

然后通过 di 添加

``` csharp
services.AddSingleton<IResponseCache, xxxResponseCache>();
```