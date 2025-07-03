# http响应压缩

网络带宽是一种有限资源。 减小响应大小通常可显著提高应用的响应速度。 减小有效负载大小的一种方式是压缩应用的响应。

通常，任何未本机压缩的响应都可以从响应压缩中获益。 未本机压缩的响应通常包括：CSS、JavaScript、HTML、XML 和 JSON。 不要压缩本机压缩的资产，例如 PNG 文件。 如果尝试进一步压缩本机压缩的响应，则大小和传输时间的任何小幅额外减少都可能被处理压缩所花费的时间所掩盖。 不要压缩小于约 150-1000 字节的文件（具体取决于文件的内容和压缩效率）。 压缩小文件的开销可能会产生比未压缩文件更大的压缩文件。

商业CDN通常都有开启响应压缩。

- 默认情况下，Brotli 压缩提供程序与 Gzip 压缩提供程序会一起添加到压缩提供程序数组中。
- 当客户端支持 Brotli 压缩数据格式时，压缩默认为 Brotli 压缩。 如果客户端不支持 Brotli，则当客户端支持 Gzip 压缩时，压缩默认为 Gzip。

## 设置

大家可以在`Metadata`中设置缓存， 具体设置项如下

- `ResponseCompression`

    是否开启响应压缩， "true" 为开启

- `ResponseCompressionMimeTypes`

    允许响应压缩中间件为压缩指定一组默认的 MIME 类型。 不填写会采用默认值 [支持的 MIME 类型的完整列表的源代码](https://github.com/dotnet/aspnetcore/blob/main/src/Middleware/ResponseCompression/src/ResponseCompressionDefaults.cs)。

    你可以填写 `application/json` , 也可以填写通配 `*/*`

- `ResponseCompressionExcludedMimeTypes`

    不允许响应压缩中间件为压缩指定一组默认的 MIME 类型。

- `ResponseCompressionEnableForHttps`

    控制安全连接上的压缩响应，该选项由于安全风险，默认处于禁用状态。 对动态生成的页面使用压缩可能会向 CRIME 和 BREACH 攻击公开该应用。 [CRIME](https://wikipedia.org/wiki/CRIME_(security_exploit)) 和 [BREACH](https://wikipedia.org/wiki/BREACH_(security_exploit)) 攻击可以通过防伪造令牌得到缓解。 有关缓解 BREACH 攻击的信息，请参阅在http://www.breachattack.com/缓解

- `ResponseCompressionLevel`

    指定用来指示压缩操作是强调速度还是强调压缩大小的值。

    - `Fastest`

        即使结果文件未可选择性地压缩，压缩操作也应尽快完成。

    - `NoCompression`

        该文件不应执行压缩。

    - `Optimal`

        压缩操作应以最佳方式平衡压缩速度和输出大小。

    - `SmallestSize`

        压缩操作应创建尽可能小的输出，即使该操作需要更长的时间才能完成。


配置示例：

``` json
{
  "ReverseProxy": {
    "Routes": {
      "a": {
        "Order": 0,  // 优先级最高，优先尝试匹配 路由 a
        "Match": {
            "Hosts": [ "api.com" ],
            "Paths": [ "*" ],
            "Statement": "Header('x-env') = 'test'"
        },
        "ClusterId": "ClusterA",
        "Metadata": {
          "ResponseCompression": "true",
          "ResponseCompressionLevel": "SmallestSize",
          "ResponseCompressionEnableForHttps": "true",
          "ResponseCompressionMimeTypes": "*/*"
        }
      }
    }
  }
}
```