# 不同监听场景如何配置

> [!WARNING]
> 虽然可以在运行时动态变更监听，但是目前[Kestrel](https://github.com/dotnet/aspnetcore/tree/main/src/Servers/Kestrel)在终止监听时会等待已有连接结束，以降低对用户影响，达到优雅关闭的效果
>
> 所以在动态变更时不一定能实时生效，一定会等待已访问连接结束(超时时间1秒，超时将强制关闭)，如果频繁变化，可能会导致问题

### 监听 UDP

这里展示如何 将 `127.0.0.1:5000` UDP 代理到 `127.0.0.1:11000`, 同时只接受服务端最多返回一个 udp 包

# [appsettings.json](#tab/udp-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : udptest
      "udptest": {
        "Protocols": [ "UDP" ], // 协议选择 UDP
        "Address": [ "127.0.0.1:5000" ], // 监听地址
        "RouteId": "udpTestRoute"  // UDP 必填路由，因为不像http 有host等协议参数能确定目的地址
      }
    },
    "Routes": {
    // Routes id : udpTestRoute
      "udpTestRoute": {
        "ClusterId": "udpTestCluster",  // udp 目的地址 id
        "UdpResponses": 1,  // 只接受服务端最多返回一个 udp 包
        "Timeout": "00:00:11"  // 超时时间 11 秒， 包含从udp包 发送到 目的地址和 等待以及返回 服务端udp包的总共时间
      }
    },
    "Clusters": {
        // Clusters id : udpTestCluster
      "udpTestCluster": {
        "Destinations": [
          {
            "Address": "127.0.0.1:11000"  //目的地址
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/udp-ui)

![listen-udp.jpg](/VKProxy.Doc/images/listen-udp.jpg)

---

### 监听 TCP

这里展示如何 将 `127.0.0.1:5000` TCP 代理到 `https://google.com`

# [appsettings.json](#tab/tcp-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : tcpTest
      "tcpTest": {
        "Protocols": [ "TCP" ], // 协议选择 TCP
        "Address": [ "127.0.0.1:5000" ], // 监听地址
        "RouteId": "tcpTestRoute"  // tcp 必填路由，因为不像http 有host等协议参数能确定目的地址
      }
    },
    "Routes": {
    // Routes id : tcpTestRoute
      "tcpTestRoute": {
        "ClusterId": "tcpTestCluster",  // tcp 目的地址 id
        "Timeout": "00:00:11"  // 超时时间 11 秒， 
      }
    },
    "Clusters": {
        // Clusters id : tcpTestCluster
      "tcpTestCluster": {
        "Destinations": [
          {
            "Address": "https://google.com"  //目的地址
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/tcp-ui)

![listen-tcp.jpg](/VKProxy.Doc/images/listen-tcp.jpg)

---

### 监听 SNI 代理 (Passthrough)

这里展示如何 将 `127.0.0.1:5000` TCP SNI 

并通过ssl握手时的 host 路由匹配代理到 `https://google.com`， 但这里我们配置不在proxy端处理ssl，而是交由后端服务自行处理

# [appsettings.json](#tab/sni-passthrough-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : tcpTest
      "tcpTest": {
        "Protocols": [
          "TCP"
        ], // 协议选择 TCP
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
        "UseSni": true // 开启 sni 代理，由于采用 公用 sni ，所以不必配置RouteId
      }
    },
    "Sni": {
      "sniPassthroughTest": {
        // 这里配置 所有 com 结尾的请求都会路由到 tcpTestRoute
        "Host": [
          "*com"
        ],
        "Passthrough": true,  // 配置不在proxy端处理ssl，而是交由后端服务自行处理
        "RouteId": "tcpTestRoute"
      },
    },
    "Routes": {
      // Routes id : tcpTestRoute
      "tcpTestRoute": {
        "ClusterId": "tcpTestCluster", // tcp 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : tcpTestCluster
      "tcpTestCluster": {
        "Destinations": [
          {
            "Address": "https://google.com" //目的地址
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/sni-passthrough-ui)

创建 listen

![listen-sni.jpg](/VKProxy.Doc/images/listen-sni.jpg)

创建sni 配置

![listen-sni-sni.jpg](/VKProxy.Doc/images/listen-sni-sni.jpg)

---

### 监听 SNI 代理 (ssl)

这里展示如何 将 `127.0.0.1:5000` TCP SNI 

并通过ssl握手时的 host 路由匹配代理到 `http://google.com`， 但这里我们配置在proxy端处理ssl，并且由于是tcp代理，所以后端服务无法使用 https， 因为已经被proxy 解密了，http请求的场景只能配置后端服务为 http了

# [appsettings.json](#tab/sni-ssl-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : tcpTest
      "tcpTest": {
        "Protocols": [
          "TCP"
        ], // 协议选择 TCP
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
        "UseSni": true // 开启 sni 代理，由于采用 公用 sni ，所以不必配置RouteId
      }
    },
    "Sni": {
      "sniPassthroughTest": {
        // 这里配置 所有 com 结尾的请求都会路由到 tcpTestRoute
        "Host": [
          "*com"
        ],
        "CheckCertificateRevocation": false, // 不进行证书校验， 因为这里配置的是自签证书，只是用于测试
        // 这里配置 PEM 格式证书， 这样配置便于更换，如果使用机器上的证书文件，有一些场景比如 docker 大家就很不方便了
        "Certificate": {
          "PEM": "-----BEGIN CERTIFICATE-----\nMIIFCzCCAvOgAwIBAgIUAi7DqcEn4EsBm1lN4UcmmuxWPq0wDQYJKoZIhvcNAQEL\nBQAwFDESMBAGA1UEAwwJbG9jYWxob3N0MCAXDTIxMDIyNjE4MzI0OVoYDzIxMjEw\nMjAyMTgzMjQ5WjAUMRIwEAYDVQQDDAlsb2NhbGhvc3QwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQDI5+DbDLpVmRFFB9YR1NwAbdPzf7cV8RB32pNVHzLj\niJ9LNSq8yvI/k6xD3p8XHVJEC3TGMuOVK4Cn277nX9KSU/7lELMlz+yGikwli3aI\nu4A7NE84ACfR6ZFelAGcGr2nMze6K2YDwhhqG8SUBIhAjzjIXSf2+cv0Cq38cEX1\nWKy1h5xJm7FwuKlZsHmJw5Osk+8mFgHJ5TmKqJYtVd7b/ytAmeM2ByZ4eVCZ31rR\nELKZ2uEWoCImLCtCfohxEnhrLm09yB7JrbbT/3JXHXyQUn3kGMczgwL/IZZwqtAf\nE4SiHVLI3cFwO3VJjpRfAbg3xFbxwzhmdlri3WsKT3iEp3P08WPi6dL7WF+PjpMl\nd9RZ587NBSqCmVfgBjY05wEcONSPyPY1gi5ZBDBm6J23feAOgZ8AZu/RKdAVgYBu\nKr6o6ZKqsJ+U+M5cwrXw/Rv78YQyr9ZlfKALbybMKMrYuc9DqbPGBbryLxklh9kN\nwMJvt8FHEcmlbT5BsZJmm7JztPouN7mxMy1ZVJlGreyPD0mxET+O4DCz7UsA2CvB\npKzLcRRHKKNbCwaeV/1UJKiyg4QWDGltlIKkIEhppv2vspl0Gh7xpZISJuWtmWiJ\no03zSr7NUNvXRY3pmNQXad7PHanVyTCopCCpTiAQljDTTb803NGyKBHx3cz17HU9\nxQIDAQABo1MwUTAdBgNVHQ4EFgQUGyQOi+8yPBZziF7ruCfQB4ooTMUwHwYDVR0j\nBBgwFoAUGyQOi+8yPBZziF7ruCfQB4ooTMUwDwYDVR0TAQH/BAUwAwEB/zANBgkq\nhkiG9w0BAQsFAAOCAgEAHf+FN6rHdZJPdmUO1skpS9iVgXrKWGwo20Qrd3MttKfk\nxzFpOZLBEyn/qWmZe1YQqdcm4Yd7OjnKRb62zwE8gyTJlaA30qXGoJZrouWEAsWZ\n2//2h/Ju6XNy47p5F2UKAKqqGcSaDy9HEQF0wNwRz45LKYlJE7v7eDqo2TOampoH\nUXNRF9lKI4o+CKkSRquoqGXfw6GJmnxrozTzWl00igSXrX3+HkiKHNOgzaOoS+pP\nnFl/HI/jOFYh8AG/18U5iFBSTjXiyXmFvkb4309c188fJd1UMOVY1tbcfFWSftnL\nYbk8UmGagtI9S8ExuQvk34TGDwj0vdKGiTBdL/qQ1vzxqLo2U7fHRcktSo27Ogtp\nJCzfyXKb41Cu4VOmzllTlhbg/p68rEeYcVIeZl86Yh3bFZNVpvHW9vzn8iLIXpGf\nnyt/XXG0cgkTPeWZ+zTPHLx/9YZBXViUuXobXLeUhueCaWGHYPkzKcV1c1B9oJjc\n/3JWbJVERFxMGgJpQUrTMerUCmY3C2lfPBm48ZmPCjmUUdWsh5vu2pVe+3hBIFeb\nY/kkOuRqAmiW+EmjFNQNdcxsDstd1AeipapPSH0TLWTqvAs8MndoNmfHyOFomV38\nEls5LL5Pomm27oVq6JM1geF1jKShAnO/w/dlRXcB0PFJIlpWKpw7OE5qqPpoiZY=\n-----END CERTIFICATE-----",
          "PEMKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----\n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\nBlK3Zix8WqtE14P/MNzfaA==\n-----END ENCRYPTED PRIVATE KEY-----",
          "Password": "testPassword"
        },
        "RouteId": "tcpTestRoute"
      },
    },
    "Routes": {
      // Routes id : tcpTestRoute
      "tcpTestRoute": {
        "ClusterId": "tcpTestCluster", // tcp 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : tcpTestCluster
      "tcpTestCluster": {
        "Destinations": [
          {
            "Address": "http://google.com" //目的地址 由于是tcp代理，所以后端服务无法使用 https， 因为已经被proxy 解密了，http请求的场景只能配置后端服务为 http了
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/sni-ssl-ui)


创建 listen

![listen-sni.jpg](/VKProxy.Doc/images/listen-sni.jpg)

创建sni 配置

![listen-sni-sni-host.jpg](/VKProxy.Doc/images/listen-sni-sni-host.jpg)

---

### 监听 http 代理

这里展示如何 将 `127.0.0.1:5000` http1 协议 ，（其他版本需要证书，所以这里单独展示配置，以便大家对比）

并通过 host 路由匹配代理到 `http://google.com`

# [appsettings.json](#tab/http-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : httpTest
      "httpTest": {
        "Protocols": [
          "HTTP1"
        ], // 协议选择 HTTP1
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
      },
    },
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        // 配置 http 请求如何匹配
        "Match": {
          "Hosts": [ "*com" ], // 匹配 host 为 com 结尾的
          "Paths": [ "/ws*" ], // url 匹配 /ws 开头的
          "Statement": "Method = 'GET'" // http method 匹配 GET， 这里提供 类似 sql 中 where 的简单表达式写法，当然有一些限制，请参见 Statement 说明
        },
        "ClusterId": "httpTestCluster", // 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "Destinations": [
          {
            "Address": "http://google.com" 
          }
        ]
      }
    }
  }
}
```

# [UI](#tab/http-ui)

监听 127.0.0.1:5000

![listen-http.jpg](/VKProxy.Doc/images/listen-http.jpg)

创建路由

![listen-http-route.jpg](/VKProxy.Doc/images/listen-http-route.jpg)

---

### 监听 https (证书通过sni匹配) 代理

这里展示如何 将 `127.0.0.1:5000` http1 / http2 协议 ，证书通过sni匹配， http3 因为目前 quic 底层实现限制， 无法在运行时动态切换证书，所以无法在sni 配置

并通过 host 路由匹配代理到 `http://google.com`

# [appsettings.json](#tab/https-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : httpTest
      "httpTest": {
        "Protocols": [
          "HTTP1","HTTP2"
        ], // 协议选择 HTTP1 HTTP2
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
        "UseSni": true // 表明 启用 https
      }
    },
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        // 配置 http 请求如何匹配
        "Match": {
          "Hosts": [ "*com" ], // 匹配 host 为 com 结尾的
          "Paths": [ "/ws*" ], // url 匹配 /ws 开头的
          "Statement": "Method = 'GET'" // http method 匹配 GET， 这里提供 类似 sql 中 where 的简单表达式写法，当然有一些限制，请参见 Statement 说明
        },
        "ClusterId": "httpTestCluster", // 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "Destinations": [
          {
            "Address": "http://google.com" 
          }
        ]
      }
    },
    "Sni": {
      "sniComTest": {
        // 这里配置 所有 com 结尾的SSL握手请求都会匹配到如下证书
        "Host": [
          "*com"
        ],
        "CheckCertificateRevocation": false, // 不进行证书校验， 因为这里配置的是自签证书，只是用于测试
        // 这里配置 PEM 格式证书， 这样配置便于更换，如果使用机器上的证书文件，有一些场景比如 docker 大家就很不方便了
        "Certificate": {
          "PEM": "-----BEGIN CERTIFICATE-----\nMIIFCzCCAvOgAwIBAgIUAi7DqcEn4EsBm1lN4UcmmuxWPq0wDQYJKoZIhvcNAQEL\nBQAwFDESMBAGA1UEAwwJbG9jYWxob3N0MCAXDTIxMDIyNjE4MzI0OVoYDzIxMjEw\nMjAyMTgzMjQ5WjAUMRIwEAYDVQQDDAlsb2NhbGhvc3QwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQDI5+DbDLpVmRFFB9YR1NwAbdPzf7cV8RB32pNVHzLj\niJ9LNSq8yvI/k6xD3p8XHVJEC3TGMuOVK4Cn277nX9KSU/7lELMlz+yGikwli3aI\nu4A7NE84ACfR6ZFelAGcGr2nMze6K2YDwhhqG8SUBIhAjzjIXSf2+cv0Cq38cEX1\nWKy1h5xJm7FwuKlZsHmJw5Osk+8mFgHJ5TmKqJYtVd7b/ytAmeM2ByZ4eVCZ31rR\nELKZ2uEWoCImLCtCfohxEnhrLm09yB7JrbbT/3JXHXyQUn3kGMczgwL/IZZwqtAf\nE4SiHVLI3cFwO3VJjpRfAbg3xFbxwzhmdlri3WsKT3iEp3P08WPi6dL7WF+PjpMl\nd9RZ587NBSqCmVfgBjY05wEcONSPyPY1gi5ZBDBm6J23feAOgZ8AZu/RKdAVgYBu\nKr6o6ZKqsJ+U+M5cwrXw/Rv78YQyr9ZlfKALbybMKMrYuc9DqbPGBbryLxklh9kN\nwMJvt8FHEcmlbT5BsZJmm7JztPouN7mxMy1ZVJlGreyPD0mxET+O4DCz7UsA2CvB\npKzLcRRHKKNbCwaeV/1UJKiyg4QWDGltlIKkIEhppv2vspl0Gh7xpZISJuWtmWiJ\no03zSr7NUNvXRY3pmNQXad7PHanVyTCopCCpTiAQljDTTb803NGyKBHx3cz17HU9\nxQIDAQABo1MwUTAdBgNVHQ4EFgQUGyQOi+8yPBZziF7ruCfQB4ooTMUwHwYDVR0j\nBBgwFoAUGyQOi+8yPBZziF7ruCfQB4ooTMUwDwYDVR0TAQH/BAUwAwEB/zANBgkq\nhkiG9w0BAQsFAAOCAgEAHf+FN6rHdZJPdmUO1skpS9iVgXrKWGwo20Qrd3MttKfk\nxzFpOZLBEyn/qWmZe1YQqdcm4Yd7OjnKRb62zwE8gyTJlaA30qXGoJZrouWEAsWZ\n2//2h/Ju6XNy47p5F2UKAKqqGcSaDy9HEQF0wNwRz45LKYlJE7v7eDqo2TOampoH\nUXNRF9lKI4o+CKkSRquoqGXfw6GJmnxrozTzWl00igSXrX3+HkiKHNOgzaOoS+pP\nnFl/HI/jOFYh8AG/18U5iFBSTjXiyXmFvkb4309c188fJd1UMOVY1tbcfFWSftnL\nYbk8UmGagtI9S8ExuQvk34TGDwj0vdKGiTBdL/qQ1vzxqLo2U7fHRcktSo27Ogtp\nJCzfyXKb41Cu4VOmzllTlhbg/p68rEeYcVIeZl86Yh3bFZNVpvHW9vzn8iLIXpGf\nnyt/XXG0cgkTPeWZ+zTPHLx/9YZBXViUuXobXLeUhueCaWGHYPkzKcV1c1B9oJjc\n/3JWbJVERFxMGgJpQUrTMerUCmY3C2lfPBm48ZmPCjmUUdWsh5vu2pVe+3hBIFeb\nY/kkOuRqAmiW+EmjFNQNdcxsDstd1AeipapPSH0TLWTqvAs8MndoNmfHyOFomV38\nEls5LL5Pomm27oVq6JM1geF1jKShAnO/w/dlRXcB0PFJIlpWKpw7OE5qqPpoiZY=\n-----END CERTIFICATE-----",
          "PEMKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----\n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\nBlK3Zix8WqtE14P/MNzfaA==\n-----END ENCRYPTED PRIVATE KEY-----",
          "Password": "testPassword"
        }
      }
    },
  }
}
```

# [UI](#tab/https-ui)


监听 127.0.0.1:5000

![listen-http-sni.jpg](/VKProxy.Doc/images/listen-http-sni.jpg)

创建路由

![listen-http-route.jpg](/VKProxy.Doc/images/listen-http-route.jpg)

---

### 监听 http3 代理

这里展示如何 将 `127.0.0.1:5000` http1 / http2 / http3 协议 ， http3 因为目前 quic 底层实现限制， 无法在运行时动态切换证书，所以特殊指定证书配置，也因此 127.0.0.1:5000 只能匹配这一证书，无法提供sni动态匹配证书的能力

并通过 host 路由匹配代理到 `http://google.com`

# [appsettings.json](#tab/http3-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : httpTest
      "httpTest": {
        "Protocols": [
          "HTTP1","HTTP2", "HTTP3"
        ], // 协议选择 HTTP1 HTTP2 HTTP3
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
        "UseSni": true, // 表明 启用 https,
        "SniId": "sniComTest" // http3 因为目前 quic 底层实现限制， 无法在运行时动态切换证书 所以特殊指定证书配置，也因此 127.0.0.1:5000 只能匹配这一证书，无法提供sni动态匹配证书的能力
      }
    },
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        // 配置 http 请求如何匹配
        "Match": {
          "Hosts": [ "*com" ], // 匹配 host 为 com 结尾的
          "Paths": [ "/ws*" ], // url 匹配 /ws 开头的
          "Statement": "Method = 'GET'" // http method 匹配 GET， 这里提供 类似 sql 中 where 的简单表达式写法，当然有一些限制，请参见 Statement 说明
        },
        "ClusterId": "httpTestCluster", // 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "Destinations": [
          {
            "Address": "http://google.com" 
          }
        ]
      }
    },
    "Sni": {
      "sniComTest": {
        // 这里配置 所有 com 结尾的SSL握手请求都会匹配到如下证书
        "Host": [
          "*com"
        ],
        "CheckCertificateRevocation": false, // 不进行证书校验， 因为这里配置的是自签证书，只是用于测试
        // 这里配置 PEM 格式证书， 这样配置便于更换，如果使用机器上的证书文件，有一些场景比如 docker 大家就很不方便了
        "Certificate": {
          "PEM": "-----BEGIN CERTIFICATE-----\nMIIFCzCCAvOgAwIBAgIUAi7DqcEn4EsBm1lN4UcmmuxWPq0wDQYJKoZIhvcNAQEL\nBQAwFDESMBAGA1UEAwwJbG9jYWxob3N0MCAXDTIxMDIyNjE4MzI0OVoYDzIxMjEw\nMjAyMTgzMjQ5WjAUMRIwEAYDVQQDDAlsb2NhbGhvc3QwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQDI5+DbDLpVmRFFB9YR1NwAbdPzf7cV8RB32pNVHzLj\niJ9LNSq8yvI/k6xD3p8XHVJEC3TGMuOVK4Cn277nX9KSU/7lELMlz+yGikwli3aI\nu4A7NE84ACfR6ZFelAGcGr2nMze6K2YDwhhqG8SUBIhAjzjIXSf2+cv0Cq38cEX1\nWKy1h5xJm7FwuKlZsHmJw5Osk+8mFgHJ5TmKqJYtVd7b/ytAmeM2ByZ4eVCZ31rR\nELKZ2uEWoCImLCtCfohxEnhrLm09yB7JrbbT/3JXHXyQUn3kGMczgwL/IZZwqtAf\nE4SiHVLI3cFwO3VJjpRfAbg3xFbxwzhmdlri3WsKT3iEp3P08WPi6dL7WF+PjpMl\nd9RZ587NBSqCmVfgBjY05wEcONSPyPY1gi5ZBDBm6J23feAOgZ8AZu/RKdAVgYBu\nKr6o6ZKqsJ+U+M5cwrXw/Rv78YQyr9ZlfKALbybMKMrYuc9DqbPGBbryLxklh9kN\nwMJvt8FHEcmlbT5BsZJmm7JztPouN7mxMy1ZVJlGreyPD0mxET+O4DCz7UsA2CvB\npKzLcRRHKKNbCwaeV/1UJKiyg4QWDGltlIKkIEhppv2vspl0Gh7xpZISJuWtmWiJ\no03zSr7NUNvXRY3pmNQXad7PHanVyTCopCCpTiAQljDTTb803NGyKBHx3cz17HU9\nxQIDAQABo1MwUTAdBgNVHQ4EFgQUGyQOi+8yPBZziF7ruCfQB4ooTMUwHwYDVR0j\nBBgwFoAUGyQOi+8yPBZziF7ruCfQB4ooTMUwDwYDVR0TAQH/BAUwAwEB/zANBgkq\nhkiG9w0BAQsFAAOCAgEAHf+FN6rHdZJPdmUO1skpS9iVgXrKWGwo20Qrd3MttKfk\nxzFpOZLBEyn/qWmZe1YQqdcm4Yd7OjnKRb62zwE8gyTJlaA30qXGoJZrouWEAsWZ\n2//2h/Ju6XNy47p5F2UKAKqqGcSaDy9HEQF0wNwRz45LKYlJE7v7eDqo2TOampoH\nUXNRF9lKI4o+CKkSRquoqGXfw6GJmnxrozTzWl00igSXrX3+HkiKHNOgzaOoS+pP\nnFl/HI/jOFYh8AG/18U5iFBSTjXiyXmFvkb4309c188fJd1UMOVY1tbcfFWSftnL\nYbk8UmGagtI9S8ExuQvk34TGDwj0vdKGiTBdL/qQ1vzxqLo2U7fHRcktSo27Ogtp\nJCzfyXKb41Cu4VOmzllTlhbg/p68rEeYcVIeZl86Yh3bFZNVpvHW9vzn8iLIXpGf\nnyt/XXG0cgkTPeWZ+zTPHLx/9YZBXViUuXobXLeUhueCaWGHYPkzKcV1c1B9oJjc\n/3JWbJVERFxMGgJpQUrTMerUCmY3C2lfPBm48ZmPCjmUUdWsh5vu2pVe+3hBIFeb\nY/kkOuRqAmiW+EmjFNQNdcxsDstd1AeipapPSH0TLWTqvAs8MndoNmfHyOFomV38\nEls5LL5Pomm27oVq6JM1geF1jKShAnO/w/dlRXcB0PFJIlpWKpw7OE5qqPpoiZY=\n-----END CERTIFICATE-----",
          "PEMKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----\nMIIJnDBOBgkqhkiG9w0BBQ0wQTApBgkqhkiG9w0BBQwwHAQItOk3T6xc6NECAggA\nMAwGCCqGSIb3DQIJBQAwFAYIKoZIhvcNAwcECAKua/2H7aHDBIIJSNpyiEmgYamY\n1U5fkmHQbCHuT16i4tN19OM9Atxyjp+zX2DmRDYf3LJEeJxDYHyATAQNmemO6FlG\nnMOUnsqVn9IMOxMQeHuesL5WhMcW/3VEWuVR4Ivr0C+MqRkia4hwufISKpKUo/oe\n/ipWE/CbbE+eG83oHNANpIuo+c+Om2+maoxNNoZnPEzQJi4Xpywsk8DLB2qBI5tW\nvuQZ+BSnomtlCNSJuFGpWzOmGsoDmYPDO6Xq9AjkPmHJO2dkeWoviuNqGsUsr/wS\nvgvZYpw4n/QnkV/PeRsh5JHpUfJVfT636uav7Xy2w00WsKmQuFjRnORKHiC9syV5\nkDIqZUi8KU3Fd3t3N/FOieAwlO+nvyHLT2KLYY74zvONh558856ZGQ553KqxbnBl\nwoIcU5wi6VzpVeupb+CZTbgeOyndf2a2md6Epb2KqhTjoOAj4zKoNus5CUy769tF\n5lLWWxpGDQZpUS0mPss/5lialNaHmpn3jLtOfX4HmP5/OTWBiKi2Skyafv0ZE+I+\nxXaB6XLsI1CT8W1nPbJnuaf3dJOH7/q2CORRlzbffXsMOe1pGt7O8ysa7KojSCUb\nbzKVLjEreQ9of7qCZjUd0TNbW5o1g6vRol63J6NfqWQOHeDhPMMJyjCsI28w4MjA\nPjvLxXcMNs7Jw2iQDh97WRya4beYK323/J/v5Zn2IqHYuY5BC1IOBPOsuhwb/R/0\nCEEz4DCDj3aQsAG8ennLAPzehHhfdOqPSYd3bVeRWraSbMIXyzku9tI+6aO3m5KJ\n/DpblQqP6IMvp1Wn7DrFEoc5rgGfxm0BCqIyFefsAm71Ib5+ShtjUk+Q/GRU301n\n1SKLLL42YaZL3eLRDabTvYPJYfYZOJ8uuEyEiQ+FWTOYmwvp712V86BKatfiGzAB\nk/4Z5Y4SQg4zKiwHoF1GC66fHBwUMQPIYV6U0ivH0kMR33of3NHPVTgaS4N3E++B\nwWMayb+D+Akfd+yYDHBhiQkRxEYy0dgbP6Z29nYCHaywR4UutU6dcePGaQpZKMf3\ni9pZiBGY79Q9Y8rCHGKbJZxYG1l5Mfc17WkbMlZnQXRu/KyFCzrFnAhyhLXYLnnQ\nLWZid2gbA2mYd5MFWFiBlwgwJrzhS0LG5waSzqR+fWcp5p2+T//B0P9Bd0XBMQpM\n9WYiU83HaERFZgxkpCKNwO+e2ve7zUiFtUFNNlcjsgAOjuECQ/on6Zi3HKp0tDOM\nD/4/hKxW01hPD7U2P9/Vhd9ninO7gXBngP+Ub858rEOphzRX+DSgP5hJ0qtQbzNB\nVbMsFk3a4YvdiqmnEG/LTeMKYEafC9iR6ul3G3xUOU72uCOw5KOSR0o27AEuqgtQ\njNSSd9K8aMohFzs39AZReHN2JkVHFTgJ9VgDENmFH7r1qQN6HGoKnm0DMtBzzdKV\nGREWawE36Ll8/KwvL+DRT0KoQuOk8v0caLInmwBzdqgxwv5ZUxw6z+vIeWCmaUAX\nhkGcYcpGKOq9FgNSelKNctf5wkbbnyvBPByQaLyYxLEe7CLXwEwp8I9ZRay/JQtS\nYEKkNW5jVwhPUIgdqFb0sWQv8wg5tZJwUnFcCeFopPznxZPJ9AwfU2m9VRiieOUg\nqcKfw9PF9rILBwxkJ8sB78jFb7gGbMJejCtOi1DnpWFghz2gTqWQNpM7z9Fk5vX4\nhBUMain1sJobHZ4xqjm102/DhxoEdkVCZrXV5ukm1tXkAbU2ot7quM0/eLUixTSo\nASgQEutXG0Jwy8nR49B4XvdOmMmtpGs9UMlN1qkLMNl9O1ORyaAhwrwNDVUw77Ws\nW6R5bup+X8WJcghG48ZTjNSLvldbYHxIgWdoXxZIBwHgtjpZsFXbodIX10bklQte\n6N3QIUofuiVLDwEk/VnzP4AmDLi/8PXa15NbPpKeabiGmgQOLLkvfa1AaIy3fIO2\n6iAm36kx6nlqzw1rWFzBslBwowiDV3XhTeyqwygkjmAnRmDOEH9rBXuZsJdw2sDn\nxahtXsYTk0ONfG43j/qwEKgy8y0tSaC9yUJn7gohHD6o/KNR8CQ2qX2pq8tyNLeu\n5+N1Wa1c44rEwyp43Vu1CwVM/9UEpzbZaPNG25Yiz/Matl/s1rOFfzKMwgOnuVyi\nYk1MPMnt19gJ0GhjAvTXD/xIxVYZsxP7aB2Pre56uZP/BSqDaHg2h1I+dgvF02RU\nlQ5WwAox0e+rnWeG6io+eGP1zEL3i8SlBJP16tk6kJxF79cCtKFdfPjQSkJL8vQ6\n/rhsQaM+a+Jw1p4XqfaD1BxvmeNfq1zm3ZoEA712YZHlqnR8MwrsWtqUe+AkM67p\nR6TCwU7/n+cQlX3PfyobDsImXaofER44pqVzW1QiKxmQOFLkDNXXlTXxo+NoXbAZ\nR6jjoZMNE/iBwFxbnzy6uuprswEbxNRUMEmJPT38nIuHOKZ6qQkyNzIb0wSdZtXm\nPKDp7XHaBpqbxvs/C0DpNfEXlY4p8IOMuxLFA1Z8fi3Aar/R8nOe+DOQZqmc+kq0\njY9BGp2CChJxEFlEVn8n1/9UqA1Xn+cFf872yoltGnvuRawjohuDpmXOjF/bQ4Xr\nnwgMMnLj0X+L0+3R9HfVsU6SUbVa4B9VBHhCPd5B6+kBtWctjvegAw3R7zOytvum\nQgVk9J/q+WXkOl3zmBAOOHpupBo81Pb/IFr9HNQcxR14Uf1BvTPZy9XNVIeudGrv\nOU6gT22brw4ed/L+K9ZpUyvhLQU4WdXK+698IKDhEb8/WCMLg1gK4cnVjZVveewg\nTp0jOfiFBv4RV4tobs6sGknb1u3IqIIccLTjKgL9IF9zkSsPopjcAiJK/UEhGihS\nth3KthUD4qW10mi3iEhhSsiJOSnJ6QxoM85xzJVCeYYUL7Fad5Kx0W87eMrzPh/q\nN6q8yEq3yYFgmGxYgZ1gib+vq1FFjoWGnu6VnLzWU7EDyaABQJMynsbbg5oyZT2m\nDqNYZXaUUpVh713tsrL6rk2ya0HBxM7OsC37rWu1DDRvTmXr63ogtVruGdLqlviw\n4rk3fNsObrGny/zUgWIWVMS07WctKe8HD1EfR0vVrdH/hiwPag4/lKRsQ+jMRuWO\nlma7Ebyu4DieZ6/hqZI0X+vb1QaL0yBwTUoe3FNPBab5GmFUyvGh+f0kAVvhvM3r\nBlK3Zix8WqtE14P/MNzfaA==\n-----END ENCRYPTED PRIVATE KEY-----",
          "Password": "testPassword"
        }
      }
    },
  }
}
```

# [UI](#tab/http3-ui)



监听 127.0.0.1:5000

![listen-http3.jpg](/VKProxy.Doc/images/listen-http3.jpg)

创建路由

![listen-http-route.jpg](/VKProxy.Doc/images/listen-http-route.jpg)

---

### gRPC 代理

gRPC 是与语言无关的高性能远程过程调用 （RPC） 框架。 它基于 HTTP/2 构建，所以是可以进行代理。 但需要确保启用正确的 HTTP 协议。 如果未正确配置为发送和接收 HTTP/2 请求，gRPC 要求 HTTP/2 和 gRPC 调用将失败。

PS: 如使用与 HTTP/1.1 兼容的 gRPC 的替代线路格式 gRPC-Web。 则无需特殊配置。

# [appsettings.json](#tab/gRPC-json)

``` json
{
  "ReverseProxy": {
    "Listen": {
      // Listen id : httpTest
      "httpTest": {
        "Protocols": [
          "HTTP2"
        ], // 协议选择 HTTP2
        "Address": [
          "127.0.0.1:5000"
        ], // 监听地址
        "UseSni": true, // 表明 启用 https
      }
    },
    "Routes": {
      // Routes id : httpTestRoute
      "httpTestRoute": {
        // 配置 http 请求如何匹配
        "Match": {
          "Hosts": [ "*com" ], // 匹配 host 为 com 结尾的
          "Paths": [ "/ws*" ], // url 匹配 /ws 开头的
        },
        "ClusterId": "httpTestCluster", // 目的地址 id
        "Timeout": "00:00:11" // 超时时间 11 秒， 
      }
    },
    "Clusters": {
      // Clusters id : httpTestCluster
      "httpTestCluster": {
        "HttpRequest": { // 确保转发请求依然使用 http2
            "Version": "2",
            "VersionPolicy": "RequestVersionExact"
        },
        "Destinations": [
          {
            "Address": "https://google.com" 
          }
        ]
      }
    },
    "Sni": {
      "sniComTest": {
        // 这里配置 所有 com 结尾的SSL握手请求都会匹配到如下证书
        "Host": [
          "*com"
        ],
        "CheckCertificateRevocation": false, // 不进行证书校验， 因为这里配置的是自签证书，只是用于测试
        // 这里配置 PEM 格式证书， 这样配置便于更换，如果使用机器上的证书文件，有一些场景比如 docker 大家就很不方便了
        "Certificate": {
          "PEM": "-----BEGIN CERTIFICATE-----\nMIIFCzCCAvOgAwIBAgIUAi7DqcEn4EsBm1lN4UcmmuxWPq0wDQYJKoZIhvcNAQEL\nBQAwFDESMBAGA1UEAwwJbG9jYWxob3N0MCAXDTIxMDIyNjE4MzI0OVoYDzIxMjEw\nMjAyMTgzMjQ5WjAUMRIwEAYDVQQDDAlsb2NhbGhvc3QwggIiMA0GCSqGSIb3DQEB\nAQUAA4ICDwAwggIKAoICAQDI5+DbDLpVmRFFB9YR1NwAbdPzf7cV8RB32pNVHzLj\niJ9LNSq8yvI/k6xD3p8XHVJEC3TGMuOVK4Cn277nX9KSU/7lELMlz+yGikwli3aI\nu4A7NE84ACfR6ZFelAGcGr2nMze6K2YDwhhqG8SUBIhAjzjIXSf2+cv0Cq38cEX1\nWKy1h5xJm7FwuKlZsHmJw5Osk+8mFgHJ5TmKqJYtVd7b/ytAmeM2ByZ4eVCZ31rR\nELKZ2uEWoCImLCtCfohxEnhrLm09yB7JrbbT/3JXHXyQUn3kGMczgwL/IZZwqtAf\nE4SiHVLI3cFwO3VJjpRfAbg3xFbxwzhmdlri3WsKT3iEp3P08WPi6dL7WF+PjpMl\nd9RZ587NBSqCmVfgBjY05wEcONSPyPY1gi5ZBDBm6J23feAOgZ8AZu/RKdAVgYBu\nKr6o6ZKqsJ+U+M5cwrXw/Rv78YQyr9ZlfKALbybMKMrYuc9DqbPGBbryLxklh9kN\nwMJvt8FHEcmlbT5BsZJmm7JztPouN7mxMy1ZVJlGreyPD0mxET+O4DCz7UsA2CvB\npKzLcRRHKKNbCwaeV/1UJKiyg4QWDGltlIKkIEhppv2vspl0Gh7xpZISJuWtmWiJ\no03zSr7NUNvXRY3pmNQXad7PHanVyTCopCCpTiAQljDTTb803NGyKBHx3cz17HU9\nxQIDAQABo1MwUTAdBgNVHQ4EFgQUGyQOi+8yPBZziF7ruCfQB4ooTMUwHwYDVR0j\nBBgwFoAUGyQOi+8yPBZziF7ruCfQB4ooTMUwDwYDVR0TAQH/BAUwAwEB/zANBgkq\nhkiG9w0BAQsFAAOCAgEAHf+FN6rHdZJPdmUO1skpS9iVgXrKWGwo20Qrd3MttKfk\nxzFpOZLBEyn/qWmZe1YQqdcm4Yd7OjnKRb62zwE8gyTJlaA30qXGoJZrouWEAsWZ\n2//2h/Ju6XNy47p5F2UKAKqqGcSaDy9HEQF0wNwRz45LKYlJE7v7eDqo2TOampoH\nUXNRF9lKI4o+CKkSRquoqGXfw6GJmnxrozTzWl00igSXrX3+HkiKHNOgzaOoS+pP\nnFl/HI/jOFYh8AG/18U5iFBSTjXiyXmFvkb4309c188fJd1UMOVY1tbcfFWSftnL\nYbk8UmGagtI9S8ExuQvk34TGDwj0vdKGiTBdL/qQ1vzxqLo2U7fHRcktSo27Ogtp\nJCzfyXKb41Cu4VOmzllTlhbg/p68rEeYcVIeZl86Yh3bFZNVpvHW9vzn8iLIXpGf\nnyt/XXG0cgkTPeWZ+zTPHLx/9YZBXViUuXobXLeUhueCaWGHYPkzKcV1c1B9oJjc\n/3JWbJVERFxMGgJpQUrTMerUCmY3C2lfPBm48ZmPCjmUUdWsh5vu2pVe+3hBIFeb\nY/kkOuRqAmiW+EmjFNQNdcxsDstd1AeipapPSH0TLWTqvAs8MndoNmfHyOFomV38\nEls5LL5Pomm27oVq6JM1geF1jKShAnO/w/dlRXcB0PFJIlpWKpw7OE5qqPpoiZY=\n-----END CERTIFICATE-----",
          "PEMKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----\n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\nBlK3Zix8WqtE14P/MNzfaA==\n-----END ENCRYPTED PRIVATE KEY-----",
          "Password": "testPassword"
        }
      }
    },
  }
}
```

# [UI](#tab/gRPC-ui)

监听 127.0.0.1:5000

![listen-http-grpc.jpg](/VKProxy.Doc/images/listen-http-grpc.jpg)

创建路由

![listen-http-grpc-route.jpg](/VKProxy.Doc/images/listen-http-grpc-route.jpg)
---