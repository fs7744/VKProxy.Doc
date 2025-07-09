# 静态内容

有很多时候我们需要临时或者不同环境添加一些不同内容，

比如 [robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt?hl=zh-cn)

在开发阶段，多半不会有人考虑这东西，多半都是遇到一些问题后才会调整这些东西，但是通常这时候调整程序就显得杀鸡用牛刀了

所以 VKProxy 就提供了一个简单的功能，可以随时添加一些静态响应内容


## 设置项

大家可以可以在cluster的`Metadata` 设置

- `xxx_Content`

    xxx 可以任意命名，只要不重复就好

    内容即为响应body

- `xxx_ContentType`

    xxx_Content 对应的 ContentType， 不设置默认为 `text/plain`

- `xxx_ContentWhen`

    xxx_Content 对应的 条件筛选 比如 `Path = '/robots.txt'` 意味着只有url为 /robots.txt才能匹配，具体配置可参考[如何为HTTP配置路由复杂匹配](/VKProxy.Doc/docs/statement)

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
          "robots_Content": "User-agent: \r\n*Allow: /\r\nSitemap: https://www.xxx.com/sitemap.xml",
          "robots_ContentType": "application/text",
          "robots_ContentWhen": "Path = '/robots.txt'"
        }
      }
    }
  }
}
```