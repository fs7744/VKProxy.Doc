# 如何为HTTP配置请求和响应转换

代理请求时，通常修改请求或响应的各个部分，以适应目标服务器的要求或流出其他数据，例如客户端的原始 IP 地址。 此过程通过Transforms实现。 为应用程序全局定义转换类型，然后各个路由提供参数以启用和配置这些转换。 原始请求对象不会由这些转换修改，只修改代理请求。

如果内置转换集不足，则可以通过[定制化扩展](/VKProxy.Doc/docs/extensibility)添加自定义转换。

( PS: 如下实现参考于 [Yarp](https://github.com/dotnet/yarp) )

## 默认行为

遵循HTTP代理公认行为，为方便大家使用，以下转换默认启用

- `Host`

    代理请求默认会修改为目标服务器地址中指定的主机名

    如需特殊修改请使用 `RequestHeaderOriginalHost`

- `X-Forwarded-For`

    将客户端的 IP 地址设置为 X-Forwarded-For 标头 （默认情况下会覆盖）

    如需特殊修改（比如多层代理情况下想保留请求中已有的值以准确传递客户端的IP）请使用 `X-Forwarded`

- `X-Forwarded-Proto`

    将请求的原始方案 (http/https) 设置为 X-Forwarded-Proto 标头。

    如需特殊修改请使用 `X-Forwarded`

- `X-Forwarded-Host`

    将请求的原始方案 (http/https) 设置为 X-Forwarded-Proto 标头。

    如需特殊修改请使用 `X-Forwarded`

- `X-Forwarded-Prefix`

    将请求的原始 PathBase 标头（如果有）设置为 X-Forwarded-Prefix 标头。 （理论上默认情况下应该没有， 除非特殊使用 PathBase 之类的中间件）

    如需特殊修改请使用 `X-Forwarded`

举例，传入 `http://IncomingHost:5000/path` 的以下请求：

``` http
GET /path HTTP/1.1
Host: IncomingHost:5000
Accept: */*
header1: foo
```

将转换并代理到目标服务器 `https://DestinationHost:6000/` ，如下所示，使用以下默认值：

``` http
GET /path HTTP/1.1
Host: DestinationHost:6000
Accept: */*
header1: foo
X-Forwarded-For: 5.5.5.5
X-Forwarded-Proto: http
X-Forwarded-Host: IncomingHost:5000
```

## 请求转换

请求转换包括请求路径、查询、HTTP 版本、方法和标头。内置转换集支持以下:

### PathPrefix

为请求路径加上给定值前缀。 

| Key | Memo |
| -- | -- |
| PathPrefix | 以“/”开头的路径 |



示例：`/request/path` 变为 `/prefix/request/path`

# [appsettings.json](#tab/PathPrefix-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { "PathPrefix": "/prefix" }
        ]
      }
    }
  }
}
```

# [UI](#tab/PathPrefix-ui)

暂未支持

---

### PathRemovePrefix

将从请求路径中删除匹配的前缀。 匹配是在路径段边界（/） 上进行的。 如果前缀不匹配，则不会进行更改。

| Key | Memo |
| -- | -- |
| PathRemovePrefix | 以“/”开头的路径 |



示例：`/prefix/request/path` 变为 `/request/path`, `/prefix2/request/path` 未修改

# [appsettings.json](#tab/PathRemovePrefix-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { "PathRemovePrefix": "/prefix" }
        ]
      }
    }
  }
}
```

# [UI](#tab/PathRemovePrefix-ui)

暂未支持

---

### PathSet

将使用给定值设置请求路径。

| Key | Memo |
| -- | -- |
| PathSet | 以“/”开头的路径 |



示例：`/request/path` 变为 `/newpath`

# [appsettings.json](#tab/PathSet-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { "PathSet": "/newpath" }
        ]
      }
    }
  }
}
```

# [UI](#tab/PathSet-ui)

暂未支持

---

### QueryValueParameter

在请求查询字符串中添加或替换参数


| Key | Memo |
| -- | -- |
| QueryValueParameter | 查询字符串参数的名称 |
| Set/Append | 静态值 |



示例：将添加一个包含名称 foo 的查询字符串参数，并将其设置为静态值 bar。即 `/request/path?a=v&foo=x` 变为 `/request/path?a=v&foo=x&foo=bar`

# [appsettings.json](#tab/QueryValueParameter-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { 
            "QueryValueParameter": "foo",
            "Append": "bar"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/QueryValueParameter-ui)

暂未支持

---


### QueryRemoveParameter

从请求查询字符串中删除指定的参数


| Key | Memo |
| -- | -- |
| QueryRemoveParameter | 查询字符串参数的名称 |



示例：如果请求中存在，这将删除具有名称 foo 的查询字符串参数。即 `/request/path?a=v&foo=x` 变为 `/request/path?a=v`

# [appsettings.json](#tab/QueryRemoveParameter-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          { 
            "QueryRemoveParameter": "foo"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/QueryRemoveParameter-ui)

暂未支持

---

### HttpMethodChange

更改请求中使用的 http 方法


| Key | Memo |
| -- | -- |
| HttpMethodChange | 需要替换的 HTTP 方法 |
| Set | 新的 http 方法 |



示例：将 PUT 请求更改为 POST。即 `PUT /request/path` 变为 `POST /request/path?a=v`

# [appsettings.json](#tab/HttpMethodChange-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "HttpMethodChange": "PUT",
            "Set": "POST"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/HttpMethodChange-ui)

暂未支持

---

### RequestHeadersCopy

这将设置是否将所有传入请求标头复制到代理请求中。 此设置默认处于启用状态，可通过将转换配置为 false 值来禁用。 即使禁用了此功能，引用特定标头的转换仍将运行。


| Key | Memo |
| -- | -- |
| RequestHeadersCopy | true/false |



示例：禁用

# [appsettings.json](#tab/RequestHeadersCopy-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "RequestHeadersCopy": "false"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/RequestHeadersCopy-ui)

暂未支持

---

### RequestHeaderOriginalHost

这指定是否应将传入请求主机标头复制到代理请求。 此设置默认处于禁用状态，可以通过配置具有 true 值的转换来启用此设置。 直接引用 Host 标头的转换将替代此转换。


| Key | Memo |
| -- | -- |
| RequestHeaderOriginalHost | true/false |



示例：启用

# [appsettings.json](#tab/RequestHeaderOriginalHost-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "RequestHeaderOriginalHost": "true"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/RequestHeaderOriginalHost-ui)

暂未支持

---

### RequestHeaderRemove

删除请求标头


| Key | Memo |
| -- | -- |
| RequestHeaderRemove | 标头名称 |



示例：

原始请求
``` http
MyHeader: MyValue
AnotherHeader: AnotherValue
```
处理后
``` http
AnotherHeader: AnotherValue
```

# [appsettings.json](#tab/RequestHeaderRemove-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "RequestHeaderRemove": "MyHeader"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/RequestHeaderRemove-ui)

暂未支持

---

### RequestHeadersAllowed

默认将大多数请求标头复制到代理请求（请参阅 RequestHeadersCopy）。 某些安全模型仅允许代理特定标头。 此转换将禁用 RequestHeadersCopy，并且仅复制给定标头。 如果未包含在允许列表中，则修改或追加到现有标头的其他转换可能会受到影响。

请注意，默认情况下，某些标头不会复制，因为它们特定于连接或其他安全敏感（例如Connection）。 Alt-Svc 将这些标头名称放在允许列表中将绕过该限制，但强烈建议不要这样做，因为它可能会对代理的功能产生负面影响或导致安全漏洞。


| Key | Memo |
| -- | -- |
| RequestHeadersAllowed | 以分号分隔的允许标头名称列表。 |



示例：

原始请求
``` http
Header1: value1
header2: value2
AnotherHeader: AnotherValue
```
处理后
``` http
Header1: value1
header2: value2
```

# [appsettings.json](#tab/RequestHeadersAllowed-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "RequestHeadersAllowed": "Header1;header2"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/RequestHeadersAllowed-ui)

暂未支持

---

### X-Forwarded

当代理连接到目标服务器时，连接与客户端对代理建立的连接是独立的。 目标服务器可能需要原始连接信息进行安全检查，并正确生成链接和重定向的绝对 URI。 若要将有关客户端连接的信息传递给目标，可以添加一组额外的标头。 在Forwarded标准建立之前，一种常见的解决方案是使用X-Forwarded-*标头。 没有定义X-Forwarded-*标头的官方标准，因此实现可能因服务器而异，请检查目标服务器是否支持。

即使路由配置中未指定，也默认启用此转换。

将 X-Forwarded 值设置为包含你需要启用的标头的逗号分隔列表。 默认情况下，所有 for 标头都处于启用状态。 可以通过指定值 "Off"来禁用所有值。

Prefix 指定要用于每个标头的标头名称前缀。 使用默认X-Forwarded-前缀时，生成的标头将为X-Forwarded-For、X-Forwarded-Proto和X-Forwarded-HostX-Forwarded-Prefix。

转换操作指定了如何将每个标头与同名的现有标头组合在一起。 它可以是“Set”、“Append”、“Remove”或“Off”（完全禁用转换功能）。 遍历多个代理的请求可能会累积此类标头的列表，目标服务器需要评估列表以确定原始值。 如果操作是“Set”并且关联的值在请求中不可用（例如 RemoteIpAddress 为 null），则仍会删除现有的任何标头以防止欺骗。

{Prefix}For 标头值取自 HttpContext.Connection.RemoteIpAddress 表示前一个调用方 IP 地址。 不包括端口。 IPv6 地址不包括方括号 []。

{Prefix}Proto 标头值取自 HttpContext.Request.Scheme 指示前一个调用方是否使用了 HTTP 或 HTTPS。

{Prefix}主机标头值取自传入请求的主机标头。 这独立于上面指定的 RequestHeaderOriginalHost。 Unicode/IDN 主机经过 punycode 编码。

{Prefix}Prefix 标头值取自 HttpContext.Request.PathBase。 生成代理请求时不使用 PathBase 属性，因此目标服务器需要原始值才能正确生成链接和重定向。 该值采用百分比编码 URI 格式。

| Key | Memo |
| -- | -- |
| X-Forwarded | 要应用于下面列出的所有 X-Forwarded* 的默认操作（Set、Append、Remove、Off） |
| For | 要应用于下面列出的所有 X-Forwarded* 的默认操作（Set、Append、Remove、Off） |
| Proto | 要应用于下面列出的所有 X-Forwarded* 的默认操作（Set、Append、Remove、Off） |
| Prefix | 要应用于下面列出的所有 X-Forwarded* 的默认操作（Set、Append、Remove、Off） |
| HeaderPrefix | 标头名称前缀 |



示例：

# [appsettings.json](#tab/X-Forwarded-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "X-Forwarded": "Set",
            "For": "Remove",
            "Proto": "Append",
            "Prefix": "Off",
            "HeaderPrefix": "X-Forwarded-"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/X-Forwarded-ui)

暂未支持

---



### Forwarded

标头 Forwarded 由 [RFC 7239](https://tools.ietf.org/html/rfc7239) 定义。 它整合了与非官方 X-Forwarded 标头相同的许多功能，将信息流向目标服务器，否则将使用代理隐藏这些信息。

启用此转换将禁用默认的 X-Forwarded 转换，因为它们以其他格式传递类似的信息。 仍可以显式启用 X-Forwarded 转换。

操作：这指定转换应如何处理现有的 Forwarded 标头。 它可以是“Set”、“Append”、“Remove”或“Off”（完全禁用转换功能）。 遍历多个代理的请求可能会累积此类标头的列表，目标服务器需要评估列表以确定原始值。

Proto：此值取自 HttpContext.Request.Scheme 指示前一个调用方是否使用了 HTTP 或 HTTPS。

主机：此值取自传入请求的主机标头。 这独立于上面指定的 RequestHeaderOriginalHost。 Unicode/IDN 主机经过 punycode 编码。

对于：此值标识以前的调用方。 IP 地址取自 HttpContext.Connection.RemoteIpAddress。 有关详细信息，请参阅下面的 ByFormat 和 ForFormat。

接收源：此值标识代理接收请求的位置。 IP 地址取自 HttpContext.Connection.LocalIpAddress。 有关详细信息，请参阅下面的 ByFormat 和 ForFormat。

ByFormat 和 ForFormat：

RFC 允许 By 和 For 字段 [的各种格式](https://datatracker.ietf.org/doc/html/rfc7239#section-6) 。 它要求默认格式使用此处标识为 Random 的经过模糊处理的标识符。


| Key | Memo |
| -- | -- |
| Forwarded | 逗号分隔列表，其中包含以下任何值：for、by、proto、host |
| ForFormat | Random/RandomAndPort/RandomAndRandomPort/Unknown/UnknownAndPort/UnknownAndRandomPort/Ip/IpAndPort/IpAndRandomPort |
| ByFormat | Random/RandomAndPort/RandomAndRandomPort/Unknown/UnknownAndPort/UnknownAndRandomPort/Ip/IpAndPort/IpAndRandomPort |
| Action | Set、Append、Remove、Off |



示例：

# [appsettings.json](#tab/Forwarded-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "Forwarded": "by,for,host,proto",
            "ByFormat": "Random",
            "ForFormat": "IpAndPort",
            "Action": "Append"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/Forwarded-ui)

暂未支持

---

### ClientCert

将入站连接上使用的客户端证书作为标头转发到目标

由于入站和出站连接是独立的，因此需要有一种方法将任何入站客户端证书传递到目标服务器。 此转换会导致从 HttpContext.Connection.ClientCertificate 中获取的客户端证书进行 Base64 编码，并设置为给定标头名称的值。 目标服务器可能需要该证书对客户端进行身份验证。 没有定义该标头的标准，不同的实现方式可能为此有所不同，请检查目标服务器是否支持。

默认情况下，服务器对传入客户端证书进行最小验证。 应在代理或目标中验证证书，有关详细信息，请参阅 客户端证书身份验证 文档。

仅当连接上已存在客户端证书时，此转换才适用。


| Key | Memo |
| -- | -- |
| ClientCert | 标头名称 |



示例：

# [appsettings.json](#tab/ClientCert-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ClientCert": "X-Client-Cert"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ClientCert-ui)

暂未支持

---

## 响应转换

默认情况下，所有响应标头和尾部都从代理响应复制到传出客户端响应。 响应和响应尾部转换可以指定它们是只应用于成功的响应还是应用于所有响应。

### ResponseHeadersCopy

此设置用于确定是否将所有代理响应标头复制到客户端响应。 此设置默认处于启用状态，可以通过配置变换为 false 值来禁用此设置。 即使禁用了此功能，引用特定标头的转换仍将运行。


| Key | Memo |
| -- | -- |
| ResponseHeadersCopy | true/false |



示例：

# [appsettings.json](#tab/ResponseHeadersCopy-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseHeadersCopy": "false"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseHeadersCopy-R-ui)

暂未支持

---

### ResponseHeader

这会设置或追加命名响应标头的值。 Set 替换任何现有标头。 Append 会添加具有给定值的额外标头。 注意：不建议将“”设置为标头值，并可能导致未定义的行为。

When 指定是否应在所有响应、成功响应或失败响应中包含响应头。 任何状态代码小于 400 的响应都被视为成功。


| Key | Memo |
| -- | -- |
| ResponseHeader | 标头名称 |
| Set/Append | 标头值 |
| When | Success/Failure/Always |



示例：

# [appsettings.json](#tab/ResponseHeader-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseHeader": "HeaderName",
            "Append": "value",
            "When": "Success"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseHeader-R-ui)

暂未支持

---

### ResponseHeaderRemove

这会删除命名的响应标头。

When 指定是否应针对所有响应、成功或失败响应删除响应标头。 任何状态代码小于 400 的响应都被视为成功。


| Key | Memo |
| -- | -- |
| ResponseHeaderRemove | 标头名称 |
| When | Success/Failure/Always |



示例：

# [appsettings.json](#tab/ResponseHeaderRemove-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseHeaderRemove": "HeaderName",
            "When": "Success"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseHeaderRemove-R-ui)

暂未支持

---

### ResponseHeadersAllowed

默认从代理响应复制大多数响应标头（请参阅 ResponseHeadersCopy）。 某些安全模型仅允许代理特定标头。 此转换将禁用 ResponseHeadersCopy，并且仅复制给定标头。 如果未包含在允许列表中，则修改或追加到现有标头的其他转换可能会受到影响。

请注意，默认情况下，某些标头不会复制，因为它们特定于连接或其他安全敏感（例如Connection）。 Alt-Svc 将这些标头名称放在允许列表中将绕过该限制，但强烈建议不要这样做，因为它可能会对代理的功能产生负面影响或导致安全漏洞。


| Key | Memo |
| -- | -- |
| ResponseHeadersAllowed | 以分号分隔的允许标头名称列表。 |



示例：

# [appsettings.json](#tab/ResponseHeadersAllowed-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseHeadersAllowed": "Header1;header2"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseHeadersAllowed-R-ui)

暂未支持

---

### ResponseTrailersCopy

此选项设置是否将所有代理响应尾部复制到客户端响应。 此设置默认处于启用状态，可以通过配置变换为 false 值来禁用此设置。 即使禁用了此功能，引用特定标头的转换仍将运行。


| Key | Memo |
| -- | -- |
| ResponseTrailersCopy | true/false |



示例：

# [appsettings.json](#tab/ResponseTrailersCopy-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseTrailersCopy": "Header1;header2"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseTrailersCopy-R-ui)

暂未支持

---
### ResponseTrailer

添加或替换尾随响应标头

响应尾部是在响应正文末尾发送的标头。 对预告片的支持在 HTTP/1.1 实现中并不常见，但在 HTTP/2 实现中很常见。 检查客户端和服务器是否支持。


| Key | Memo |
| -- | -- |
| ResponseTrailer | 标头名称 |
| Set/Append | 标头值 |
| When | Success/Failure/Always |



示例：

# [appsettings.json](#tab/ResponseTrailer-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseTrailer": "HeaderName",
            "Append": "value",
            "When": "Success"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseTrailer-R-ui)

暂未支持

---

### ResponseTrailerRemove

删除尾随响应标头

When 指定是否应针对所有响应、成功或失败响应删除响应标头。 任何状态代码小于 400 的响应都被视为成功。


| Key | Memo |
| -- | -- |
| ResponseTrailerRemove | 标头名称 |
| When | Success/Failure/Always |



示例：

# [appsettings.json](#tab/ResponseTrailerRemove-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseTrailerRemove": "HeaderName",
            "When": "Success"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseTrailerRemove-R-ui)

暂未支持

---

### ResponseTrailersAllowed

默认从代理响应复制大多数响应预告片（请参阅 ResponseTrailersCopy）。 某些安全模型仅允许代理特定标头。 此转换将禁用 ResponseTrailersCopy，并且仅复制给定标头。 如果未包含在允许列表中，则修改或追加到现有标头的其他转换可能会受到影响。

请注意，默认情况下，某些标头不会复制，因为它们特定于连接或其他安全敏感（例如Connection）。 Alt-Svc 将这些标头名称放在允许列表中将绕过该限制，但强烈建议不要这样做，因为它可能会对代理的功能产生负面影响或导致安全漏洞。


| Key | Memo |
| -- | -- |
| ResponseTrailersAllowed | 以分号分隔的允许标头名称列表。 |



示例：

# [appsettings.json](#tab/ResponseTrailersAllowed-R-json)

``` json
{
  "ReverseProxy": {
    "Routes": {
      "xxxRoute": {
        "Transforms": [
          {
            "ResponseTrailersAllowed": "Header1;header2"
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/ResponseTrailersAllowed-R-ui)

暂未支持

---