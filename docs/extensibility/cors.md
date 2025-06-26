# 跨域（CORS）设置

跨源资源共享（CORS，或通俗地译为跨域资源共享）是一种基于 HTTP 头的机制，该机制通过允许服务器标示除了它自己以外的其他源（域、协议或端口），使得浏览器允许这些源访问加载自己的资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的“预检”请求。在预检中，浏览器发送的头中标示有 HTTP 方法和真实请求中会用到的头。

跨源 HTTP 请求的一个例子：运行在 https://domain-a.com 的 JavaScript 代码使用 XMLHttpRequest 来发起一个到 https://domain-b.com/data.json 的请求。

出于安全性，浏览器限制脚本内发起的跨源 HTTP 请求。例如，XMLHttpRequest 和 Fetch API 遵循同源策略。这意味着使用这些 API 的 Web 应用程序只能从加载应用程序的同一个域请求 HTTP 资源，除非响应报文包含了正确 CORS 响应头。

![fetching-page-cors.svg](https://mdn.github.io/shared-assets/images/diagrams/http/cors/fetching-page-cors.svg)

CORS 机制允许 Web 应用服务器进行跨源访问控制，从而使跨源数据传输得以安全进行。现代浏览器支持在 API 容器中（例如 XMLHttpRequest 或 Fetch）使用 CORS，以降低跨源 HTTP 请求所带来的风险。

#### 什么情况下需要 CORS？
这份跨源共享标准允许在下列场景中使用跨站点 HTTP 请求：

- 前文提到的由 XMLHttpRequest 或 Fetch API 发起的跨源 HTTP 请求。
- Web 字体（CSS 中通过 @font-face 使用跨源字体资源），因此，网站就可以发布 TrueType 字体资源，并只允许已授权网站进行跨站调用。
- WebGL 贴图。
- 使用 drawImage() 将图片或视频画面绘制到 canvas。
- 来自图像的 CSS 图形。

更详细描述可以参见 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS

## 如何在 VKProxy 设置 CORS

在某些情况，大家可能不想修改已有程序设置跨域，而像api gateway 就有功能可以在代理设置 CORS

# [appsettings.json](#tab/json)

通过`appsettings.json`配置

``` json
{
  "ReverseProxy": {
    "Routes": {
      "b": {
        "Order": 0, 
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ]
        },
        "ClusterId": "ClusterB",
        // Metadata 中设置相关 header 值
        "Metadata": {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "POST,PUT"
        }
      }
    },
  }
}
```

# [ui](#tab/ui)

通过`ui`配置

![cors.jpg](/VKProxy.Doc/images/cors.jpg)

---

### 具体配置项列举

- `Access-Control-Allow-Origin`

    参数指定了单一的源，告诉浏览器允许该源访问资源。或者，对于不需要携带身份凭证的请求，服务器可以指定该字段的值为通配符“*”，表示允许来自任意源的请求。

- `Access-Control-Allow-Origin-Regex`

    与 `Access-Control-Allow-Origin` 作用一致，不过通过正则表达式让大家能有更加灵活或者复杂的控制能力。

- `Access-Control-Allow-Headers`

    标头字段用于预检请求的响应。其指明了实际请求中允许携带的标头字段。这个标头是服务器端对浏览器端 Access-Control-Request-Headers 标头的响应。

- `Access-Control-Allow-Methods`

    标头字段指定了访问资源时允许使用的请求方法，用于预检请求的响应。其指明了实际请求所允许使用的 HTTP 方法。

- `Access-Control-Allow-Credentials`

    指定了当浏览器的 credentials 设置为 true 时是否允许浏览器读取 response 的内容。当用在对 preflight 预检测请求的响应中时，它指定了实际的请求是否可以使用 credentials。请注意：简单 GET 请求不会被预检；如果对此类请求的响应中不包含该字段，这个响应将被忽略掉，并且浏览器也不会将相应内容返回给网页。

- `Access-Control-Max-Age`

    指定了 preflight 请求的结果能够被缓存多久，

- `Access-Control-Expose-Headers`

    在跨源访问时，XMLHttpRequest 对象的 getResponseHeader() 方法只能拿到一些最基本的响应头，Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma，如果要访问其他头，则需要服务器设置本响应头。