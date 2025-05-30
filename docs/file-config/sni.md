# Sni

TLS 证书设置都是统一到 SNI 下面配置，这样方便公用

当然对于 TCP ， 我们还特殊支持了 SNI 代理模式， 便于大家复用端口，节约资源


SNI同样每一个id必须唯一，不能重复，因为运行时会根据配置变动动态调整监听，不必重启实例

SNI 有以下配置项

- `Order`

    int 类型

    优先级，数字越小优先级越高， 您可以使用优先级在比如 路由冲突场景确定顺序，避免冲突

    默认路由匹配优先顺序为 1）Host (如冲突则会按照 Order依次匹配)


- `Host`

    string[] 类型

    域名匹配列表， 可以配置以下格式

    - 精确匹配

        完全按照host一模一样对比，例如`Hosts:["a.com"]` 只能匹配 a.com ，不能匹配 aa.com

    - 后缀匹配

        后缀一致的匹配方式，例如`Hosts:["*a.com"]` 能匹配 aa.com ，不能匹配 ab.com

    - 任意

        `Hosts:["*"]` 这样任意域名都会匹配上，无论 aa.com 还是 bb.org

- `Passthrough`

    当 代理 tcp SNI 模式时，可选择何时处理证书验证

    当值为 true 不在proxy端处理TLS，而是交由后端服务自行处理，此时 `Certificate`配置将无效，所以也不必配置

    当值为 false 在proxy端处理TLS，而是转发给后端服务的请求会是解密之后的， 所以 tcp 后端服务不能同时使用TLS 并且 `Certificate`必须配置

- `Protocols`

    启用的 [SSL 协议](https://learn.microsoft.com/zh-cn/dotnet/api/system.security.authentication.sslprotocols?view=net-9.0)。 协议名称指定为字符串数组。 默认值为 None

- `CheckCertificateRevocation`

    指定在身份验证期间是否检查证书吊销列表。

- `ClientCertificateMode`

    配置[客户端证书要求](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.aspnetcore.server.kestrel.https.clientcertificatemode?view=aspnetcore-9.0) 如下列举所有的值

    - `AllowCertificate`

        将请求客户端证书;但是，如果客户端未提供证书，身份验证不会失败。

    - `DelayCertificate`

        客户端证书不是必需的，并且不会在连接开始时从客户端请求。 应用程序稍后可能会请求它。

    - `NoCertificate`

        客户端证书不是必需的，也不会从客户端请求。

    - `RequireCertificate`

        将请求客户端证书，并且客户端必须提供有效的证书才能成功进行身份验证。

- `HandshakeTimeout`

    指定 TLS/SSL 握手所允许的最长时间。 这必须为正或 InfiniteTimeSpan。 默认值为 10 秒。

- `Certificate`

    证书源

    （一些测试证书如何生成可参考 [在 PowerShell 中创建证书](https://learn.microsoft.com/zh-cn/aspnet/core/security/authentication/certauth?view=aspnetcore-9.0#create-certificates-in-powershell) [生成并导出证书 - Linux - OpenSSL](https://learn.microsoft.com/zh-cn/azure/vpn-gateway/point-to-site-certificates-linux-openssl)）

    可以将证书节点配置为从多个源加载证书：

    - `Path` 和 `Password` 用于加载 .pfx 文件。
    - `Path`、`KeyPath` 和 `Password` 用于加载 .pem/ 和 .key 文件。
    - `PEM`、`PEMKey` 和 `Password` 直接从.pem 内容用于加载。
    - `Subject` 和 `Store` 用于从证书存储中加载。

    证书源所有配置属性如下

    - `Path` 
        
        是证书文件的路径和文件名，关联包含应用内容文件的目录。
    - `KeyPath` 
    
        是证书密钥文件的路径和文件名，关联包含应用内容文件的目录。
    - `PEM` 
    
        是PEM格式证书文件内容
    - `PEMKey` 

        是PEM格式证书密钥文件内容
    - `Password`
    
        是访问 X.509 证书数据所需的密码。
    - `Store` 
    
        是从中加载证书的证书存储。
    - `Subject` 
    
        是证书的主题名称。
    - `AllowInvalid` 
    
        指示是否存在需要留意的无效证书，例如自签名证书。
    - `Location` 
    
        是从中加载证书的存储位置。

- `RouteId`

    路由配置对应 id

    只有当使用 tcp sni代理时，需要直接明确代理到哪一个目的地址，所以必须配置 RouteId，

    其他 https 证书场景不必指定