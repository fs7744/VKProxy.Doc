# ACME协议client

ACME 协议是一种开放标准，旨在实现数字证书颁发和续订流程的自动化，它彻底改变了证书管理。ACME 的开发旨在简化整个流程，已被许多证书颁发机构 (CA) 广泛采用，并已成为互联网标准 ([RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)).

为了充分理解ACME协议考虑如何让代理更方便使用ACME协议申请TLS/SSL证书，本人照着[LettuceEncrypt](https://github.com/natemcmaster/LettuceEncrypt) 和 [certes](https://github.com/fszlin/certes) 从0实现了一遍 ACME协议client以及如何在asp.net core 中集成ACME申请管理。

本文接下来会详细说明相关内容。

## ACME协议内容

这里简要描述一下协议内容，方便大家理解，详细还是得看协议原文。

一个简要ACME协议申请流程大致如下

``` shell
client  -- 0. 申请账号 -->                      ACME服务器
    |    <-- 账号信息 --                            |
    
    |    -- 1. 创建证书申请订单 -->                  |
        <-- 订单信息 --  

    |   -- 2. 选择http/dns/tls中任意之一验证方式 -->  |
        <-- 返回验证信息

    |  -- 3. 部署验证信息
        <---|

    |  -- 4. 告知可以进行验证行为                     |
             异步进行验证，成功则标明订单可以生成证书 --|                 
                                       | -->

    |  -- 4.1 client 可轮询api 确认验证结果

    |  -- 5.验证通过提交CSR最终确定订单 -->            |

    |  -- 6. 下载证书                 -->            |
```

不同验证行为如下

- http

    这是当今最常见的验证方式。 ACME服务器 如Let’s Encrypt 向您的 ACME 客户端提供一个令牌，然后您的 ACME 客户端将在您对 Web 服务器的 http://<你的域名>/.well-known/acme-challenge/<TOKEN>（用提供的令牌替换 <TOKEN>）路径上放置指定文件。 该文件包含令牌以及帐户密钥的指纹。 一旦您的 ACME 客户端告诉 ACME服务器 如Let’s Encrypt 文件已准备就绪，ACME服务器 如Let’s Encrypt 会尝试获取它（可能从多个地点进行多次尝试）。 如果我们的验证机制在您的 Web 服务器上找到了放置于正确地点的正确文件，则该验证被视为成功，您可以继续申请颁发证书。 如果验证检查失败，您将不得不再次使用新证书重新申请。

    即 需要提供 `GET http://{申请域名}/.well-known/acme-challenge/{验证Token}` api ，并返回 `{验证Token}.{AccountKey.Thumbprint()}`

    ACME服务器会确认返回是否一致

    优点：

    - 它可以轻松地自动化进行而不需要关于域名配置的额外知识。
    - 它允许托管服务提供商为通过 CNAME 指向它们的域名颁发证书。
    - 它适用于现成的 Web 服务器。
    - 它也可以用于验证 IP 地址。
   
   缺点：

    - 如果您的 ISP 封锁了 80 端口，该验证将无法正常工作（这种情况很少见，但一些住宅 ISP 会这么做）。
    - ACME服务器 如Let’s Encrypt 不允许您使用此验证方式来颁发通配符证书。
    - 您如果有多个 Web 服务器，则必须确保该文件在所有这些服务器上都可用。

- dns

    此验证方式要求您在该域名下的 TXT 记录中放置特定值来证明您控制域名的 DNS 系统。它允许您颁发通配符证书。 在 ACME服务器 如Let’s Encrypt 为您的 ACME 客户端提供令牌后，您的客户端将创建从该令牌和您的帐户密钥派生的 TXT 记录，并将该记录放在 _acme-challenge.<YOUR_DOMAIN> 下。 然后 ACME服务器 如Let’s Encrypt 将向 DNS 系统查询该记录。 如果找到匹配项，您就可以继续颁发证书！

    优点：

    - 您可以使用此验证方式来颁发包含通配符域名的证书。
    - 即使您有多个 Web 服务器，它也能正常工作。
    - 即使服务器不对公网开放，您也可以通过此方式验证其域名。

    缺点：

    - 在 Web 服务器上保留 API 凭据存在风险。
    - 您的 DNS 提供商可能不提供 API。
    - 您的 DNS API 可能无法提供有关更新时间的信息。
    - IP 地址不能通过此方式验证。
    - vkproxy lib 默认不提供dns自动验证实现，因为DNS 提供商太多了，api 不统一，且可能不提供 API

- tls

    通过 443 端口上的 TLS 执行。 但是，它使用自定义的 ALPN 协议来确保只有知道此验证类型的服务器才会响应验证请求。 这还允许对此质询类型的验证请求使用与要验证的域名匹配的SNI字段，从而使其更安全。

    这一验证类型并不适合大多数人。 它最适合那些想要执行类似于 HTTP-01 的基于主机的验证，但希望它完全在 TLS 层进行以分离关注点的 TLS 反向代理的作者。

    优点：

    - 它在 80 端口不可用时仍可以正常工作。
    - 它可以完全仅在 TLS 层执行。
    - 它也可以用于验证 IP 地址。

    缺点：

    - 它不支持 Apache、Nginx 和 Certbot，且很可能短期内不会兼容这些软件。
    - 与 HTTP-01 一样，如果您有多台服务器，则它们需要使用相同的内容进行应答。
    - 此方法不能用于验证通配符域名。

## pebble 本地测试的acme服务

由于过于贫穷，所以无法在真实的域名/acme服务商/dns服务商等等进行真实的示例

不过好在 letsencrypt.org 开源提供了可以在本地测试的acme服务 [pebble](https://github.com/letsencrypt/pebble)


配置示例
``` json
{
  "pebble": {
    "listenAddress": "0.0.0.0:14000",
    "managementListenAddress": "0.0.0.0:15000",
    "certificate": "D:\\Program Files\\tool\\pebble\\certs\\localhost\\cert.pem",
    "privateKey": "D:\\Program Files\\tool\\pebble\\certs\\localhost\\key.pem",
    "httpPort": 80,
    "tlsPort": 443,
    "ocspResponderURL": "",
    "externalAccountBindingRequired": false,
    "domainBlocklist": ["blocked-domain.example"],
    "retryAfter": {
        "authz": 3,
        "order": 5
    },
    "profiles": {
      "default": {
        "description": "The profile you know and love",
        "validityPeriod": 7776000
      },
      "shortlived": {
        "description": "A short-lived cert profile, without actual enforcement",
        "validityPeriod": 518400
      }
    }
  }
}

```

启动命令

``` shell
.\pebble.exe -config .\pebble-config.json
```

## 命令行

安装命令行

``` shell
dotnet tool install --global VKProxy.Cli
```

``` sh
// 测试ACME服务访问协议
vkproxy acme terms --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// data:text/plain,Do what thou wilt
```

``` sh
// 生成pem格式账号密钥到 accountkey 文件，acme协议默认是通过密钥关联账号
vkproxy acme account key --algorithm ES256 --output accountkey --format pem

// accountkey 文件内容：
// -----BEGIN EC PRIVATE KEY-----
// MHcCAQEEIKAVlrieijZYRLawRZhNCTfQU++umiQD1TcFCv1POgQboAoGCCqGSM49
// AwEHoUQDQgAEPhWnyoiS01MdguO4NA/4RmXO0GEFAg7a9F08KwjILDZPNNNy8XTj
// WyUU6j4IZUbt/SM53QJXNEEYqdfDU53jag==
// -----END EC PRIVATE KEY-----
```

``` sh
// 新建account账号
vkproxy acme account new --contact mailto:test@t.org --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// Location: https://127.0.0.1:14000/my-account/359a2c1419c73551
// Account: {"status":"valid","contact":["mailto:test@t.org"],"orders":"https://127.0.0.1:14000/list-orderz/359a2c1419c73551"}
```

``` sh
// 新建订单
vkproxy acme order new --domains kubernetes.docker.internal --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// https://127.0.0.1:14000/my-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA
// {"status":"pending","expires":"2025-07-27T07:09:34+00:00","identifiers":[{"type":"dns","value":"kubernetes.docker.internal"}],"authorizations":["https://127.0.0.1:14000/authZ/BmLiNkioTcxg1KCmJrHYiapFyDZAUo87w0wP8WwQYP8"],"finalize":"https://127.0.0.1:14000/finalize-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA"}
```

``` sh
// 使用 http 方式验证
vkproxy acme order authz --domain kubernetes.docker.internal --order https://127.0.0.1:14000/my-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA --challenge-type http --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// {"location":"https://127.0.0.1:14000/chalZ/t0NRkPk5MiJuc5TQkaf6u9fAZzLV4K9Kwy0ApvrhULs","challengeUri":".well-known/acme-challenge/aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0","challengeTxt":"aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0.z1pFvCHE8G1C_w6FrgJqy-YK2cUpLAFgtFzMx4bKsjg","resource":{"type":"http-01","url":"https://127.0.0.1:14000/chalZ/t0NRkPk5MiJuc5TQkaf6u9fAZzLV4K9Kwy0ApvrhULs","status":"pending","token":"aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0"}}

// 这里就要求我们部署一个处理challengeUri 的api `GET http://kubernetes.docker.internal/.well-known/acme-challenge/aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0` 返回 challengeTxt `aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0.z1pFvCHE8G1C_w6FrgJqy-YK2cUpLAFgtFzMx4bKsjg`
// 比如 部署一个 asp.net core 程序， 它包含如下内容 
// app.Map("/.well-known/acme-challenge", mapped =>
// {
//     mapped.Use(async (HttpContext c, Func<Task> next) =>
//     {
//         string value = "aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0.z1pFvCHE8G1C_w6FrgJqy-YK2cUpLAFgtFzMx4bKsjg";
//         c.Response.ContentLength = value?.Length ?? 0;
//         c.Response.ContentType = "application/octet-stream";
//         await c.Response.WriteAsync(value);
//         await c.Response.CompleteAsync();
//     });
// });
```

``` sh
// 部署好服务后，告知acme验证
vkproxy acme order validate --domain kubernetes.docker.internal --order https://127.0.0.1:14000/my-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA --challenge-type http --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir
// output:
// {"location":"https://127.0.0.1:14000/chalZ/t0NRkPk5MiJuc5TQkaf6u9fAZzLV4K9Kwy0ApvrhULs","resource":{"result":{"type":"http-01","url":"https://127.0.0.1:14000/chalZ/t0NRkPk5MiJuc5TQkaf6u9fAZzLV4K9Kwy0ApvrhULs","status":"processing","token":"aHBa1xzb32c_VvS5Lsi8s6pIB7JeyqHxIQ4C490jDH0"},"id":1,"status":"ranToCompletion","isCanceled":false,"isCompleted":true,"isCompletedSuccessfully":true,"creationOptions":"none","isFaulted":false}}
// 这里可以看到 "status":"processing"
```

``` sh
// 通过list查看
vkproxy acme order list --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// https://127.0.0.1:14000/my-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA
// {"status":"ready","expires":"2025-07-27T07:09:34+00:00","identifiers":[{"type":"dns","value":"kubernetes.docker.internal"}],"authorizations":["https://127.0.0.1:14000/authZ/BmLiNkioTcxg1KCmJrHYiapFyDZAUo87w0wP8WwQYP8"],"finalize":"https://127.0.0.1:14000/finalize-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA"}
// 这里可以看到 "status":"ready" 说明验证成功，证书可以下载了
```

``` sh
// 下载证书，因为 pebble是本地测试服务，无合法根证书，所以要多添加 --additional-issuer issuer.txt ，issuer.txt内容来自 pebble 服务 https://127.0.0.1:15000/roots/0 

vkproxy acme order finalize --algorithm ES256 --format pem --output cert --additional-issuer issuer.txt --domain kubernetes.docker.internal --order https://127.0.0.1:14000/my-order/KP92OBeu8Loim2er5K_ugWLtjGsdntMMzL28mDPhbmA --challenge-type http --key accountkey --dangerous-certificate true --timeout 00:10:00 --server https://127.0.0.1:14000/dir

// output:
// cert.pem
// -----BEGIN CERTIFICATE-----
// MIICYDCCAUigAwIBAgIIT4B4lP9vtcQwDQYJKoZIhvcNAQELBQAwKDEmMCQGA1UE
// AxMdUGViYmxlIEludGVybWVkaWF0ZSBDQSA0NzA1OTUwHhcNMjUwNzI2MDczMzQ2
// WhcNMjUwODAxMDczMzQ1WjAAMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAExpcZ
// fAZJgCeZ6iXBWmGUvwzq+RqmtUQG8jO2JEpIzTPmBHWQdLvWBiCrZQ5ssF64e44D
// UbiVbMExvpX5GIUNDaOBgDB+MA4GA1UdDwEB/wQEAwIHgDATBgNVHSUEDDAKBggr
// BgEFBQcDATAMBgNVHRMBAf8EAjAAMB8GA1UdIwQYMBaAFOAW+aaPdbX8v+N58YWB
// w5Umg3luMCgGA1UdEQEB/wQeMByCGmt1YmVybmV0ZXMuZG9ja2VyLmludGVybmFs
// MA0GCSqGSIb3DQEBCwUAA4IBAQBWylL5NhRQzJ/m7n7GUhyKEM0jvybH5uNkRu9V
// NR2hQdz/rPc8bw+9N3z3iNHkn65V9W6iC9xlwXXD7jAiYmtf3LLhYvkenbfJA72d
// f8j5brIM+3IYAnLCMkkyIsFfSMfj9pwrPt/qMjkxFq2QmoFsgPQgx/xImU+OiKKP
// t1lWaAMP/qiNWWbtRSyZ51C3RGNIVH0q+JoSRVgkbRXoxWueQted3YkBV8VDbbIW
// o0Jk6Y6xBeNFx1Lz5yqa3xnotE9m7VFTxlkaHLRkGDoO0dgj+3FHK+0XLoNt8jgN
// b9RgzCxAIBxkAlvx5VJOpApFTJhXR6hvDwyKmVvyXbZbx/7A
// -----END CERTIFICATE-----
// -----BEGIN PUBLIC KEY-----
// MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAExpcZfAZJgCeZ6iXBWmGUvwzq+Rqm
// tUQG8jO2JEpIzTPmBHWQdLvWBiCrZQ5ssF64e44DUbiVbMExvpX5GIUNDQ==
// -----END PUBLIC KEY-----
// -----BEGIN PRIVATE KEY-----
// MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQg6efLajrFYCnWy4i3
// YW5Mc1L1h04oWat/786OQEh+1JihRANCAATGlxl8BkmAJ5nqJcFaYZS/DOr5Gqa1
// RAbyM7YkSkjNM+YEdZB0u9YGIKtlDmywXrh7jgNRuJVswTG+lfkYhQ0N
// -----END PRIVATE KEY-----
这就是证书所有内容了
```

## asp.net core 中使用

只需配置好 acme 相关设置即可启动， 如

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddOpenApi();

builder.Services.AddAcmeChallenge(o =>
{
    o.AllowedChallengeTypes = VKProxy.ACME.AspNetCore.ChallengeType.Http01;
    o.RenewDaysInAdvance = TimeSpan.FromDays(2);
    o.Server = new Uri("https://127.0.0.1:14000/dir");
    o.DomainNames = new[] { "kubernetes.docker.internal" };
    o.NewAccount(new string[] { "mailto:test@xxx.com" });
    o.AdditionalIssuers = new[] {"""
            -----BEGIN CERTIFICATE-----
            MIIDGzCCAgOgAwIBAgIIU3M7k6+spYMwDQYJKoZIhvcNAQELBQAwIDEeMBwGA1UE
            AxMVUGViYmxlIFJvb3QgQ0EgMDYyYzdjMCAXDTI1MDcyNjA3MDA1MVoYDzIwNTUw
            NzI2MDcwMDUxWjAgMR4wHAYDVQQDExVQZWJibGUgUm9vdCBDQSAwNjJjN2MwggEi
            MA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDZKIeNyaVFwuVOSc+3Q3bSznnf
            QLDtUHnpwwzY6VaCW5x/M+zK4ykrIUJvC8qE55TL7YmBnJ5uT0DDsjLoZAWudRGS
            UKvivcoEWectl2YfhUSCqw1LbTuK52UQTWNNwfe+1rmFPs2C3yyfEA78221SsQsj
            FfbTZkhLgDpajtSLs9yZy+wEael8xvdMAO+REm9I8sCoK31DEs3ZNQBcrSDyT9mz
            URhzDRahov7bg2MJmBxZH8ICfINd1yZA9kNghLtaRSRLF3JZWcjCr4H1MdjlJFDY
            pQzfa7ZHCHW1fzwdRvi/zjASKvYAkr+arweQSYIqKrs9wN+ah09uEhztOz59AgMB
            AAGjVzBVMA4GA1UdDwEB/wQEAwIChDATBgNVHSUEDDAKBggrBgEFBQcDATAPBgNV
            HRMBAf8EBTADAQH/MB0GA1UdDgQWBBSH6Q9bP8CGt5JpTCMMNZj4j/DiqDANBgkq
            hkiG9w0BAQsFAAOCAQEAryVZdW8KihxLrh4yRuLbIXpjyWacoblvUrWwIQ5vnwwt
            RDoo0mHlYVOxo0ueiUQ4vi5kkGZk7VEsDXi6GV+KT/maupq6Hr+o6drKDO8iYA33
            XuDCNOgfPOXusmiPJFCm07Ah+yV3BxLWMl3azbuiGIWyRZI+fzdnGD1Rh1vPXtI8
            3JgSyqOrNLBQUVMfdhEAYNZrlFBuqUbxXEvA24IL2UgNpYTwAn2iYCcg2zpw5E/c
            DtjJTHO5x+uyXsaRQDXkJ9OZbeil691JcJH7TNxAJVe5N46JFdIf7ELvyJek/K5/
            xted2WWSLd/WQ2UPxxdfceRE1IDH0X88kk/OmmzujA==
            -----END CERTIFICATE-----

            """
};
}, c =>
{
    c.HttpClientConfig = new VKProxy.Config.HttpClientConfig()
    {
        DangerousAcceptAnyServerCertificate = true
    };
});

var app = builder.Build();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

打开debug log， 可以看到相关申请证书的log
``` shell
info: VKProxy.ACME.AspNetCore.AcmeState[0]
      Using account https://127.0.0.1:14000/my-account/43cc0ec8ef818d32
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      Creating new order for a certificate
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      Validate Http01 for kubernetes.docker.internal
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      GetAuthorization kubernetes.docker.internal
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      GetAuthorization kubernetes.docker.internal
dbug: VKProxy.ACME.AspNetCore.HttpChallengeResponseMiddleware[0]
      Confirmed challenge request for vnEH5-HvtTiFvkygIBO6Njmbu6YY7-l9DXyrCwwmSMU
dbug: VKProxy.ACME.AspNetCore.HttpChallengeResponseMiddleware[0]
      Confirmed challenge request for vnEH5-HvtTiFvkygIBO6Njmbu6YY7-l9DXyrCwwmSMU
dbug: VKProxy.ACME.AspNetCore.HttpChallengeResponseMiddleware[0]
      Confirmed challenge request for vnEH5-HvtTiFvkygIBO6Njmbu6YY7-l9DXyrCwwmSMU
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      GetAuthorization kubernetes.docker.internal
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      Creating cert for kubernetes.docker.internal
warn: VKProxy.ACME.AspNetCore.ServerCertificateSelector[0]
      Failed to validate certificate for  (AD6C43DE80E1E3FB8975F0FC2EE2E545FC42DD10). This could cause an outage of your app.
dbug: VKProxy.ACME.AspNetCore.AcmeState[0]
      Checking certificates' renewals for kubernetes.docker.internal
```

证书已经被加载到 asp.net core 中， 所以https 请求将会看到使用的 pebble的证书  

Certificate CN
Issuer CN
Pebble Intermediate CA 470595

比如请求
```
curl --location 'https://localhost:443/WeatherForecast' \
--header 'Host: kubernetes.docker.internal'
```

不过在 asp.net core 这样使用证书，个人并不推荐，这种方式存在一些问题

- 实例需要访问ACME 服务，存在额外网络维护和安全的成本
- ACME 服务通常存在一些限流，以避免攻击或滥用，当实例很多或反复启动容易产生问题
- 这样使用就会导致同一域名存在很多证书，一旦某一实例无法更新证书，实例就会产生问题，人工处理可能比较麻烦

合理做法可以是有单独程序提供证书管理的功能，证书更新则可以在变更后由管理程序调用 代理程序api进行更新。

## client with code

当然除了上述使用方式，你还可以在代码中自行进行证书申请

### 简化用法

如果已经集成好完全的自动证书申请验证，就可以使用已经封装好的代码进行简单使用

举例在asp.net core提供 一个api  可以根据参数申请证书

starup
``` csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddAcmeChallengeCore(config: c =>
{
    c.HttpClientConfig = new VKProxy.Config.HttpClientConfig()
    {
        DangerousAcceptAnyServerCertificate = true
    };
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}
app.UseAuthorization();

app.MapControllers();

app.Run();
```

api
``` csharp
using Microsoft.AspNetCore.Mvc;
using VKProxy.ACME.AspNetCore;
using VKProxy.Core.Extensions;

namespace WithApi.Controllers;

[ApiController]
[Route("[controller]")]
public class CertController : ControllerBase
{
    private readonly IAcmeStateIniter initer;

    public CertController(IAcmeStateIniter initer)
    {
        this.initer = initer;
    }

    [HttpGet]
    public async Task<string> Get([FromQuery] string domain)
    {
        // 证书配置
        var o = new AcmeChallengeOptions()
        {
            AllowedChallengeTypes = VKProxy.ACME.AspNetCore.ChallengeType.Http01,
            Server = new Uri("https://127.0.0.1:14000/dir"),
            DomainNames = new[] { domain },
            AdditionalIssuers = new[] { """
                        -----BEGIN CERTIFICATE-----
                        MIIDGzCCAgOgAwIBAgIIUPFry5qBu34wDQYJKoZIhvcNAQELBQAwIDEeMBwGA1UE
                        AxMVUGViYmxlIFJvb3QgQ0EgMjFjNjY3MCAXDTI1MDcyMjAxMTA0OVoYDzIwNTUw
                        NzIyMDExMDQ5WjAgMR4wHAYDVQQDExVQZWJibGUgUm9vdCBDQSAyMWM2NjcwggEi
                        MA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCxNKa4y93OFYaSx8bcbuWsHHnW
                        mpfsobK5Elf7GE02mi/cDrMP+wR1l53BuucrW04OyoewkBsJNZoxEy1DkCjxv4+g
                        Q+HgGCR5R14ex17ZdFxpcl42H8QnRB3IqVBlJiz0JyGZwiaOamOkUTVEYTGDeuxu
                        PglpvboGeatsWQe0MJJfBN8OxLVUmi6Y/enbzlIdv3tvgQujfPNiS8MLDMBuIiMs
                        ixhu8YAzUqvVKZoQVK7GwbD9WrVBKub8w86StKFmU14aSXahidt8IENdpLO2OT3J
                        y1nt25QDsAmtS1/wGnTDPeefLGsM7kGYNesQkSW0w8Um4p9KLWKnKyOvzPZrAgMB
                        AAGjVzBVMA4GA1UdDwEB/wQEAwIChDATBgNVHSUEDDAKBggrBgEFBQcDATAPBgNV
                        HRMBAf8EBTADAQH/MB0GA1UdDgQWBBRoXcwo6c5J8jMweiHKPw4OlcWIQzANBgkq
                        hkiG9w0BAQsFAAOCAQEAad9XT4sN1KserYtCxBKmoPhPAHInHYgG/Z2gd6KqdsK9
                        biIgEbKo84tClLqA6XCN/yN1bMQL2ZMbWBF8oHv/A5o0atpTpd+Ho+punHYRIpqv
                        akUX21Zsu6NdAuH7g7m9t9h/lc6tgiqaAf2HwpC3NrXmUlPRqLay7/t+BFQU6dBa
                        E+qzmL7lHZQf1UArfb+QDYH2XsFCk9Pjv0xdP+PGwf8HqHhfPLctvus5JL+LXp0X
                        68eWKQCs1CrL8cUMwcELlW/mR1lKnJL1WgM1Bns9ZF1ha6egG539ruzQjItF6MHB
                        xAEt55nXfs+mjV1p7qrcmR8jIdByR9C36T21r+8pKA==
                        -----END CERTIFICATE-----

                        """
            }
        };

        // 申请全新account 
        o.NewAccount(new string[] { "mailto:test11@xxx.com" });
        // 执行全套流程
        var cert = await initer.CreateCertificateAsync(o);
        return cert.ExportPem();
    }
}
```

默认情况下三种验证方式：

#### http

如在 asp.net core 中使用，默认已经添加了 `app.Map("/.well-known/acme-challenge"` 路由处理， 如要申请公网上权威认证的证书，请将 `/.well-known/acme-challenge` 路由暴露在公网让acme服务器可以访问

其次由于默认实现没有持久化和分布式处理验证信息，重启和多实例都会有问题，如有需求可以替换`IHttpChallengeResponseStore`实现以达到效果

``` csharp
public interface IHttpChallengeResponseStore
{
    Task AddChallengeResponseAsync(string token, string keyAuth, CancellationToken cancellationToken);

    Task<string> GetChallengeResponse(string token, CancellationToken cancellationToken);

    Task RemoveChallengeResponseAsync(string token, CancellationToken cancellationToken);
}
```

#### dns

由于不同服务商有各自的api，所以默认没有实现，该功能其实无效，如需使用，请实现 `IDnsChallengeStore`

``` csharp
public interface IDnsChallengeStore
{
    Task AddTxtRecordAsync(string acmeDomain, string dnsTxt, CancellationToken cancellationToken);

    Task RemoveTxtRecordAsync(string acmeDomain, string dnsTxt, CancellationToken cancellationToken);
}
```

#### tls

个人并不推荐使用tls验证方式，其由于验证自签证书和正式证书都会在 tls 层，运行时不停机重新申请对于tls管理还是有些挑战的

如想尝试可以实现`ITlsAlpnChallengeStore` （默认在 asp.net core 的实现并不能支持过滤验证自签证书只用于acme服务器请求）

``` csharp
public interface ITlsAlpnChallengeStore
{
    Task AddChallengeAsync(string domainName, X509Certificate2 cert, CancellationToken cancellationToken);

    Task RemoveChallengeAsync(string domainName, X509Certificate2 cert, CancellationToken cancellationToken);
}
```

### 底层 client

如需直接使用原始 acme 协议client，可参考如下

starup
``` csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

builder.Services.AddACME(c =>
{
    c.HttpClientConfig = new VKProxy.Config.HttpClientConfig()
    {
        DangerousAcceptAnyServerCertificate = true
    };
});
```

use
``` csharp
var context = services.BuildServiceProvider().GetRequiredService<IAcmeContext>();

await context.InitAsync(new Uri("https://127.0.0.1:14000/dir"), cancellationToken);

var account = await context.NewAccountAsync(new string[] { "mailto:xxx@xxx.com" }, true, KeyAlgorithm.RS256.NewKey());

var order = await context.NewOrderAsync(new string[] { "test.com" });
var aus = order.GetAuthorizationsAsync().ToBlockingEnumerable().ToArray();
var a = aus.First();
var b = await a.HttpAsync();
var c = await b.ValidateAsync();

Key privateKey = KeyAlgorithm.RS256.NewKey();
var csrInfo = new CsrInfo
{
    CommonName = "test.com",
};
order = await context.FinalizeAsync(csr, key, cancellationToken);
var acmeCert = await order.DownloadAsync();

var pfxBuilder = acmeCert.ToPfx(privateKey);
if (!string.IsNullOrWhiteSpace(Args.AdditionalIssuer) && File.Exists(Args.AdditionalIssuer))
{
    pfxBuilder.AddIssuer(File.ReadAllBytes(Args.AdditionalIssuer));
}
var pfx = pfxBuilder.Build("HTTPS Cert - " + Args.Domain, string.Empty);
var r = X509CertificateLoader.LoadPkcs12(pfx, string.Empty, X509KeyStorageFlags.Exportable);
```

### ui

在 管理站点的 ui sni 里面添加了 简单的 http 验证方式的acme证书界面配置 

当然使用前提得是 暴露 xxx域名/.well-known/acme-challenge 接口到公网，这样公网acme 才能验证

![acme.jpg](/VKProxy.Doc/images/acme.jpg)