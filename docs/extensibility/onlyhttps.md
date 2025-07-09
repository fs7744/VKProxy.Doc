# 禁止http

有一些原因需要我们禁用 HTTP（即只允许 HTTPS，不允许明文 HTTP），比如：

- 数据加密安全

HTTP 传输的数据是明文的，容易被窃听、篡改和中间人攻击。而 HTTPS 使用 TLS/SSL 加密，能保护数据在传输过程中的安全性和隐私性。

- 身份验证

HTTPS 通过证书机制可以验证服务器身份，防止被钓鱼网站冒充。HTTP 无法保证你访问的是“真实”的服务器。

- 数据完整性

通过 HTTPS 传输的数据无法被中途篡改，而 HTTP 没有任何防护，容易被劫持或修改内容。

- 合规要求

许多法律法规（如GDPR、PCI DSS等）要求必须加密用户敏感数据的传输，明文 HTTP 不符合这些合规要求。

- 浏览器政策

现代主流浏览器（如 Chrome、Firefox）对 HTTP 网站会高亮“不安全”，甚至屏蔽部分功能，如获取地理位置、摄像头、麦克风等。部分浏览器或 API 也要求强制使用 HTTPS。

- SEO 优势

搜索引擎（如 Google）会优先收录和排名 HTTPS 网站，禁用 HTTP 有助于提升网站权重。

所以VKProxy 提供了非常简单的强制重定向功能

## 设置项

大家可以可以在cluster的`Metadata` 设置`OnlyHttps` 为 `"true"` 来启用

配置示例：

``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,  
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ],
            "Statement": "Header('x-env') = 'test'"
        },
        "ClusterId": "ClusterA",
        "Metadata": {
          "OnlyHttps": "true"
        }
      }
    }
  }
}
```

## 实现其实非常简单

``` csharp
public class OnlyHttpsFunc : IHttpFunc
{
    public int Order => -1000;

    public RequestDelegate Create(RouteConfig config, RequestDelegate next)
    {
        if (config.Metadata == null || !config.Metadata.TryGetValue("OnlyHttps", out var v) || !bool.TryParse(v, out var b) || !b) return next;
        return c =>
        {
            if (c.Request.IsHttps)
            {
                return next(c);
            }
            else
            {
                c.Response.Redirect($"https://{c.Request.Host}{c.Request.GetEncodedPathAndQuery()}", true);
                return c.Response.CompleteAsync();
            }
        };
    }
}
```