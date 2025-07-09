# 简单的waf

WAF 是一种专门用于保护 Web 应用程序的安全防护。它通过检测、过滤和拦截进出 Web 应用的 HTTP/HTTPS 流量，防止常见的 Web 攻击，比如：

- SQL 注入（SQL Injection）
- 跨站脚本攻击（XSS, Cross-Site Scripting）
- 文件包含漏洞
- 远程命令执行
- 等等

VKProxy 肯定无法做到专业级别的防护（毕竟俺不是吃这一口饭的，没必要没钱还去撞个头破血流），只提供基本功能：用户可以设置基本的匹配规则限制相应的请求。

这样没钱没精力的用户可以优先不修改程序的场景临时做一些简单的处理， 

比如 wordpress 搭建的站点都有管理页面 yourdomain.com/wp-admin， 你并不想暴露这些地址到外网

## 设置项

大家可以可以在cluster的`Metadata` 设置

- `xxx_waf`

    xxx 可以任意命名，只要不重复就好

    配置内容即为条件筛选 比如 `Path = '/robots.txt'` 意味着只有url为 /robots.txt才能匹配，具体配置可参考[如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)

    只有匹配就会直接返回 403， 不再转发到后端服务器

配置示例：

``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,  
        "Match": {
            "Hosts": [ "*.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "ClusterA",
        "Metadata": {
          "noadmin_waf": "Path = '/wp-admin'"
        }
      }
    }
  }
}
```