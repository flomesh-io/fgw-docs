---
title: "HTTP/HTTPS 负载均衡"
description: "L7 负载均衡功能可以对更高级别的网络协议（如 HTTP、HTTPS）进行有效的负载均衡。本文档将介绍如何使用 FGW 的 L7 负载均衡功能。"
weight: 2
---

## 介绍

负载均衡是确保应用程序的可用性和弹性的关键组件。它能够分发进入的网络和应用程序流量到后端服务器，这确保了用户请求能够得到快速处理，同时避免了单一服务器的过载。HTTP/HTTPS 负载均衡特指对基于 HTTP 和 HTTPS 协议的流量进行分发。有如下常见的使用场景:

- 保障高流量应用程序的稳定性和响应速度。
- 分发流量到多个服务器以提高性能和容错能力。
- 提供 SSL/TLS 终结，从而集中管理证书和解密流量。

FGW 的 L7 负载均衡功能可以基于网络请求的内容，如路径、头信息、方法等进行流量分配，提供精细化的负载均衡机制。

如果要负载均衡 TCP 流量，请参考文档 [L4 负载均衡](/features/tcp-load-balancer)。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务

## HTTP 配置

### 配置监听端口

要使用 L7 负载均衡，首先我们需要将监听端口的协议设置为 `HTTP`，参考文档 [监听端口配置](/reference/configuration/#2-监听端口配置listeners)。

- `Port`：应用使用的端口号。这是设置服务将接收请求的端口。有效范围是 1~65535，并且是必须配置的。
- `Listen`：监听端口号。这不是必须的配置项。如果未配置，FGW 将使用 `Port` 字段值作为监听端口号。有效范围同样是 1~65535。
- `Protocol`：端口所使用的协议。这定义了服务将如何处理入站请求。可选值包括 `HTTP`、`HTTPS`、`TLS`、和 `TCP`，这是一个必须配置的字段。
- `AccessControlLists`：访问控制列表。此字段用于设置访问者 IP 地址的黑白名单。若设置了白名单，则只允许白名单中的 IP 访问；若未设置白名单，但设置了黑名单，则除黑名单外的所有 IP 都可访问。具体的设置方法和格式可以参考 [访问控制规则文档](/reference/configuration/#21-accesscontrollists)。
- `bpsLimit`：网络限速。该字段用于限制服务的网络带宽使用，单位为字节/秒。例如，如果设置了 `"bpsLimit": 10000000`，那么网络的最大吞吐量将被限制为 10MB/s。
- `TLS`：配置 TLS 相关的证书信息。此字段用于设置和管理端到端的加密连接，确保数据在传输过程中的安全性。具体的设置方法和格式可以参考 [TLS 配置文档](/reference/configuration/#22-tls)。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080
    }
  ]
}
```

### 配置端口的路由规则

接下来是设置端口 `8080` 的路由规则，参考文档 [HTTP 协议端口号路由规则配置](/reference/configuration/#31-端口号配置protocol-为-httphttps-的配置格式)。端口的路由规则是一个键值对，键为域名，值为路由规则。这里我们使用 `*` 来匹配所有的域名，也就是说忽略域名。路由规则的字段：

- `RouteType`：路由规则，可选 `HTTP` 或者 `GRPC`。
- `Matches`：匹配规则，可根据请求路径、头部、参数或者方法等信息将请求匹配到响应的后端。
  - `Path`：HTTP uri path 匹配。当请求的 URI 路径与此字段匹配时，将根据其他匹配条件确定是否路由到指定的后端服务或静态页面目录。这不是一个必须字段，但常用于基于 URL 的路由策略。具体匹配规则请参考 [Path 规则配置](/reference/configuration/#31111-path)。
  - `Headers`：HTTP header 匹配。您可以基于请求头中的特定字段进行路由。例如，可以根据 `User-Agent` 头信息来路由移动设备的请求到特定的后端服务。具体匹配规则请参考 [头部规则配置](/reference/configuration/#31112-headers)。
  - `Methods`：允许的 HTTP 方法。这允许你基于请求的 HTTP 方法（例如 `GET`、`POST`、`DELETE` 或 `PUT`）来进行路由选择。当请求方法与此字段中的任何方法匹配时，将考虑其他匹配条件。
  - `QueryParams`：HTTP 请求参数匹配。可以根据 URL 查询参数中的特定键值对进行路由，这提供了更细粒度的匹配能力。具体匹配规则请参考 [请求参数规则配置](/reference/configuration/#31113-queryparams)。
  - `BackendService`：后端服务。这定义了将请求路由到哪个后端服务。值为一个键值对，其中键是服务名，值是权重。与 `ServerRoot` 相比，`BackendService` 或 `ServerRoot` 必须至少有一个存在。
  - `ServerRoot`：静态页面所对应的目录。如果请求匹配并且需要路由到静态内容，则使用此字段指定的目录作为静态内容的根目录。与 `BackendService` 相比，`BackendService` 或 `ServerRoot` 必须至少有一个存在。
  - `Method`：当 `RouteType` 为 `GRPC` 时，用于匹配**服务**的字段。这允许根据特定的 gRPC 方法进行路由选择。具体匹配规则请参考 [GRPC 服务规则配置](/reference/configuration/#31114-method)。
  - `RateLimit`：路由限流配置。可以对匹配的路由进行限流，以防止某些请求因频繁访问而影响系统性能。具体限流规则请参考 [限流使用文档](/features/policies/rate-limiting/)。

```json
{
  "RouteRules": {
    "8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": 100
            }
          }
        ]
      }
    }
  }
}
```

### 配置服务端点

参考文档 [服务配置](/reference/configuration/#4服务配置services)。

- `StickyCookieName`：使用 cookie sticky 负载均衡时的 cookie 名称。该字段用于在 sticky 负载均衡策略中标识客户端，以确保客户端请求始终被路由到同一上游服务。使用方法可参考文档 [会话保持](/features/policies/session-sticky/)。
- `StickyCookieExpires`：使用 cookie sticky 时的 cookie 有效期。该字段定义了 cookie 的生命周期，以秒为单位。例如，如果设置为 3600，那么 cookie 的有效期为 1 小时。
- `HealthCheck`：对上游服务的健康检查配置。健康检查用于定期检查上游服务的健康状态，确保请求只被路由到健康的服务实例。具体的使用方法，可参考文档 [健康检查](/features/policies/healthcheck/)。
- `Endpoints`：上游服务信息。这是必须配置的字段，它定义了 FGW 将请求路由到哪些上游服务。具体的配置方法和格式可以参考 [端点配置文档](/reference/configuration/#412-endpoints)。
- `Filters`：过滤器配置。过滤器用于在请求被路由到上游服务之前或之后执行特定的操作，如修改请求或响应、记录日志等。具体的配置方法和格式可以参考 [过滤器配置文档](/reference/configuration/#413-filters)。
- `CircuitBreaking`：熔断配置。熔断器用于在上游服务出现问题时防止请求流向该服务，以此来保护服务和提高系统的可用性。此配置仅适用于 `HTTP` 和 `HTTPS` 协议的场景。具体的使用方法可参考文档 [熔断](/features/policies/circuit-breaking/)。
- `RetryPolicy`：重试配置。此字段定义了当请求失败时的重试策略，如重试次数、重试间隔等。此配置同样只适用于 `HTTP` 和 `HTTPS` 协议的场景。具体的使用方法可参考文档 [重试](/features/policies/retry/)。
- `UpstreamCert`：访问上游时默认使用的 TLS 证书。当 FGW 与上游服务之间的通信需要加密时，此字段用于提供 TLS 证书的配置。具体的使用方法，可以参考文档 [TLS 上游](/features/tls/tls-upstream/)。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  }
}
```

### 配置插件链

不要忘记插件链的配置，本文档主要介绍 HTTP 流量的负载均衡，这里只需要引入 `HTTPRoute` 插件链和其中的部分插件即可。完整的插件配置可以参考文档 [完整的插件配置](/reference/plugin/#完整配置)，这里我们使用的是 HTTP 负载均衡的最小插件集合。

```json
{
  "Chains": {
    "HTTPRoute": [
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

### 完整配置

最后完整的配置如下，使用其替换 FGW 工程中的 `pjs/config.json`。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080
    }
  ],
  "RouteRules": {
    "8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": 100
            }
          }
        ]
      }
    }
  },
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  },
  "Chains": {
    "HTTPRoute": [
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

使用地址 `http://localhost:8080` 多次访问负载均衡器的 `8080` 端口，可以看到轮流返回两个后端服务的响应。

## HTTPS 配置

1. 要让负载均衡器支持 HTTPS 配置，我们需要先签发 CA 证书和服务器证书。

```shell
openssl genrsa 2048 > ca-key.pem

openssl req -new -x509 -nodes -days 365000 \
   -key ca-key.pem \
   -out ca-cert.pem \
   -subj '/CN=flomesh.io'

openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server.csr -subj '/CN=example.com'
openssl x509 -req -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -days 365
```

执行上面的命令，我们会得到如下几个文件。

```
ca-cert.pem
ca-cert.srl
ca-key.pem
server-cert.pem
server-key.pem
server.csr
```

> 可以使用命令 `awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' ca-cert.pem` 来输出单行。

2. 接下来在前面 [HTTP 配置](#http-配置) 的基础上，先添加一个新的端口 `8443`，协议类型为 `HTTPS`。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080
    },
    {
      "Protocol": "HTTPS",
      "Port": 8443,
      "TLS": {}
    }
  ]
}
```

3. TLS 证书的配置，可以参考文档 [TLS 配置](/reference/configuration/#22-tls)，将上面生成的 `server-cert.pem`、`server-key.pem`、`ca-cert.pem` 分别配置在 `CertChain`、`PrivateKey`、`IssuingCA` 字段。

```json
{
  "TLS": {
    "TLSModeType": "Terminate",
    "mTLS": false,
    "Certificates": {
      "CertChain": "-----BEGIN CERTIFICATE-----\nMIICpzCCAY8CCQDlrcc1+3rp8zANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMB4XDTIzMDgwNzA3NDU0MVoXDTI0MDgwNjA3NDU0MVowFjEUMBIG\nA1UEAwwLZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIB\nAQC5OKtPgkgiUlzRWgFFyuNrm+oD5pWHaEyHQyfM/GkW3wJvqnQGPOC7SZr2du8a\nQGoC8qFJnkNkqcXwpCe9RrLAyndsny/U75TQQMTf9MojE46/lhf6YtMzip+hWnEy\ny/azYfaLsnbFu33ZHWaEIeOz8kNpK/qP5CNeMrjH9Ph3RLj0a3gXQcyvmVf9HX1A\nP2HuZsbf5q9X0cVqxcn6Irtw4F2CRU/NUcRMIofk+OgxDYc6Y+XBZfGt54C1x1lM\nxkHnigtIyItw7Ps+MgfczSK175b6xY/eM47Nnl7l7VH6PqMGBEi/NQVDO0toGSgk\nxtFZm2RSLJOO+iMzbjCurmkjAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAD9PVwvV\nJBDqfv1MSs13t309su/so+MNdnYqpIx8zsCUEu9uLVEZYwmK+KLxZJ33MDR5VuZk\ncXiNpIJqnhndMf/Ep/E/LiDvRvygNcXUhrTvTJOwo5rJtt9mxmCtGV/S9VIOXZ9d\nBnvVfb2RgnSNmw6odxH+C56Dxu2HX+byhDqeR03JONfJ8414uw0jOWNNb590haQn\n8AE+BrFeWOscv7/8hRf5oiWICfbPt+0Y+d9Ed8qyWD6lWamdWGraY4c+VfVYUve+\nXapZWJNWou2wsFwn3e0yZG7PyRBo3k2J7IocEzUN/zI96okdtJ+2smhgw68/bYLj\nKWFfharluQhLAtI=\n-----END CERTIFICATE-----\n",
      "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----MIIEowIBAAKCAQEAwVHWlokixQtr2cM3OFeqJ1Y2YXMwTEP8tKxPD47IzUJqB17tMOsy6fssDEOaTAz05eEdlFRkQHveVazouxG6goLFzooRWmyiGL+sMq6z2ZsfjOP3gnCQbLW37uRJJGSK1IIyvaUsbiI29Ltx/WJxJm3mYFhRMoM/XZ5RYGwUjkYjjIrLs8/cUHuTbVK4W8vPYOJ7cxoLhozfqv8wGPa9fEPtsn0KGLm2gpPgwpKH2wh35VZ0IHkcIVlj4siKwpNeH68VVFLO2wfaMHD0UGgPJcZEigLSf6SI2PY3j8JEmpungb+e6XbUS7mrNjj7jne0+Kjh1jCa8acBSeWcQDujFwIDAQABAoIBAHyhkTGtqZ/VNCvJAjGtusHvf9GlnG6eqi2kpLfH+sbx2T91QH94MnPMfWJOtwvukngdgJ9fJN65vOYJOmVYEaEQRAxa0MM2I+7Gq3JlVQemTVnconYSsdmT8cfunwT6WNKWObYv5Yv/POTcb6nGrGH1Gj/k0Dw7hz+I0LfUFhB+Io0DV4T9UBzXDtKkbp1s9jM94GxxsEjePJAsjBDNbpTcEvVJusgUVgwyA0kmGhc0+DwpnUJH3r1sIxLWeVhB+lmPAACCngphdHkWpGsdjPe6aE68Lb24jNv3U195nwkGgTLv4TX+fhgq2IpAz1jrz8qzumFZAm7UQ7LFUoBMCWECgYEA4isHTF56qGMemU8WUPvIyP00U2tIv9/7eRKUiiVS3izfBdasOWiGY+k1fQzrJXI77CjPaWhlQrwgdr0sDqk2a+5zzMIuNikB05gG2MoMs3pBpggHu306gC3axokwwWlJn4VwFO9GwsBhbRP9YM1Oq1P/Vi6v+oxvvbfaT1/GLkcCgYEA2tGf3+mIF41Fo8K2eTa0NJBiIsHacLK/4IAjLKwORcdVr7/211q4ECWB8m9g/sdl86LHDY4pD82/VjqSeu5qK1NgzbusgNEY6BW6AssYw0cWXm7CNf1m0CDjNKLzs7WdtQRCeY1SKYKEXceIelJ6pHRO3yWykP6Kmkl+Qz07PLECgYEAsUR2gPYgf3DJH/KsFCd09Yv4glW5fKKq8PeOM0UT0Y4r8+CRtqFljFPSl8QTXpNNwkkuYHjxvT/E1ixppsgcHraUTu332H2Fr/odi7e6AsaVQ/RRUzPRMXw/WJNZAo9qpDyrX803khfFhQBA/amNup2oqT0Is4F1Z6b91m7D36sCgYAp0AKXu70omvMirrNFiEF5BdnqwFYoUM+/a1zNTXdQuB1Ufv8A+bHQTAp/s+654IpHuuQEYBTSk0Mri/evi903uC/4QBNfbhUvS++GVx69Odk5ZDqyLGC4BoDD7xtYTKz9CPpW1b1Md0cp0FXw4c/TmvHzS/XKJQmBH+gDmzC1kQKBgD/9KpDsXs19pkuj3MmIAfWpCEiuQHd9TwrAf0cvH6cfeI1nmRBv49nhSxrBmVH5/cKWqE1xlN7dC+E1kH7qZUDrwbVe1p1m+YYbGShKv8olYlmbT5Z5zpTDnKkotjpkx9TuXC3bqrmvniREAm7W3FyXZn9x1Vc2gxu4tqni8U4J-----END RSA PRIVATE KEY-----\n",
      "IssuingCA": "-----BEGIN CERTIFICATE-----\nMIICqDCCAZACCQClfWE0dDDQ/DANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMCAXDTIzMDgwNzA3NDU0MVoYDzMwMjIxMjA4MDc0NTQxWjAVMRMw\nEQYDVQQDDApmbG9tZXNoLmlvMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC\nAQEA1tmBTZgxEb8NdlavWK+//KqdTdCrpwD841MXgjm2/tlhkuS1wNdqFfNwJtMv\nSdHmHI8V5LcqY56WZKgrlTTMlmROgWqV4+LjsrQNfnW2D/exbvtGPb9yjGIXsWVx\nGR0jddeAUlwC1o48A2u9st2pciFnzLYMd5MYQk3USmMoO/+pJgHEIn1wOC27de6Q\nU0x2SVZoja+ov6xrKOZBIChwhXzY1KvS73XRfZ1gTqMHde1By9EoByPnaMz3wN/9\nM3tSosW1q1KdN/TXkXa0MbyhUhKIjs+1q0YZJVknUBmNV2219yyteNLmlGaGqCL4\nf8R9/QE5DmXpaplXR03oPQL1/wIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQA1+wx1\npg+nZZAnbkdV3Lqe+FnWf3TegnkrTVhytxPLajRgtQ6iY+S2KYwPwXlPp6bpwbMd\nxmHy9wrpvozPdacgC+ZwqjoqiWmwk5RuimAbVL/GkNNC5HaWuWlyeA6V2sUr5poj\nDrlxfU2C0Yk80krGOK4+MUcwDlAqQq62/8xv7Ofdo/4uN3oB6WPG76xkevAgfNu9\n6saKaXZRLBAIXmvEidY6vKsnwjAhcidDPpfeJhgFOGGtnD+uTld5X5cOqzPR06Wa\n+mLu5uaEhpSbCngxRRiEnKOMScz7oyX8dXc3ZIkHLrsKN4hNdqe1h/vr1qDwFf9f\n/+ECh0PElzLMepoK\n-----END CERTIFICATE-----\n"
    }
  }
}
```

4. 接下来为 HTTPS 端口 `8443` 配置路由规则，可以在原来的键 `8080` 上添加 `8443`（使用逗号 `,` 间隔），复用 HTTP 端口的路由配置。

```json
{
  "RouteRules": {
    "8080,8443": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": 100
            }
          }
        ]
      }
    }
  }
}
```

6. HTTPS 场景下，多了 TLS 卸载的逻辑。需要为 HTTPS 增加一个新的 [HTTPSChain](/reference/plugin/#https-路由)，跟 [前面 `HTTPChain`](/features/http-load-balancer/#http-配置) 的唯一区别就是引入了 `common/tls-termination.js` 插件。

```json
{
  "Chains": {
    "HTTPSRoute": [
      "common/tls-termination.js",
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

7. 最终完整的配置如下：

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080
    },
    {
      "Protocol": "HTTPS",
      "Port": 8443,
      "TLS": {
        "TLSModeType": "Terminate",
        "mTLS": false,
        "Certificates": [
          {
            "CertChain": "-----BEGIN CERTIFICATE-----\nMIICpzCCAY8CCQDlrcc1+3rp9DANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMB4XDTIzMDgwNzA4NTA0NVoXDTI0MDgwNjA4NTA0NVowFjEUMBIG\nA1UEAwwLZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIB\nAQDWjG2AJ99p4dmG2yczeG76r4veV+6/nSYIA/UcPIA/VPE24/74/qCfz0+bGbQI\no2MEGZon71NluU2L85kkuJmfZLhfZRxOoCbjzP+8BdVth/hm/OHXf4Lf8EPQJbdy\n5Gf9XWuFYpEZllN6LBQzrdAjMutlAGIXf3d1QcJbFM9cofOcA745Bm5+7M7c9cPT\nyMROwjdymBhm3IYMnF4JkIoTvsqsSYf0DyIpsRtMumuikNRAx0JYu3OIiDNt8tJT\nT79zDcFcXxCiyndNHNRN8LmwxhWBMdYB+asfaiuJJ9C5gvRCIIcH54cyUE8WZmcm\nsA4oVqnq0QO6CGAAD8F+p+QfAgMBAAEwDQYJKoZIhvcNAQELBQADggEBADq4pUQE\nVbgjmBh/bQqQ8jshpTQDK+3lbFKoLYsBxCHokoAQ7JiyqTQfyGA8AYVm2ero9X5i\n44ezy14kTXmYD5k8vN2aTTySnIgbHtCUafhCFckn6jvszVPxqUzFsly1bbOf7PZE\nKQk9flqJA7+9qhVlw8c1/TPbyuXWS2SVTXp01mCA+3LjQHaQNMjm0RpZQZJVgPwM\nt8vGpLbXUbd5Ptn3i7aXgIIHp5Lub68hyrwEvDLTEeHnWz9KPRLS+xI/uGDzdR67\nIqneMn9Skx8sNE8hKEI0Im19+BBdWnOJZNQGe+s9EaAwbBrqaYNU+YyljeSF0tNZ\nUlevnb0vrsHIfho=\n-----END CERTIFICATE-----\n",
            "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEowIBAAKCAQEA1oxtgCffaeHZhtsnM3hu+q+L3lfuv50mCAP1HDyAP1TxNuP+\n+P6gn89Pmxm0CKNjBBmaJ+9TZblNi/OZJLiZn2S4X2UcTqAm48z/vAXVbYf4Zvzh\n13+C3/BD0CW3cuRn/V1rhWKRGZZTeiwUM63QIzLrZQBiF393dUHCWxTPXKHznAO+\nOQZufuzO3PXD08jETsI3cpgYZtyGDJxeCZCKE77KrEmH9A8iKbEbTLpropDUQMdC\nWLtziIgzbfLSU0+/cw3BXF8Qosp3TRzUTfC5sMYVgTHWAfmrH2oriSfQuYL0QiCH\nB+eHMlBPFmZnJrAOKFap6tEDughgAA/BfqfkHwIDAQABAoIBAQCtCpH+vSoKgigq\nBnP1pXsNIa0T5aQgU6Uq7dYxsfJWIjJy7SzmsqfmfRRdqjt0hCMGWYfmEbcX4n7T\nE+Q+o8zzrA6wkiJkn/L95IeWpLXhI7uLhQa6ApQR/f0T0nfFaMceqMxhxn/1PTOS\n5B5fGB85ZIZK7iYvgZVds24IfB5LPLDfruh7aezM9Gbu/i1tryZyasKZUXiiCEvy\n+18D2Ae7cF37VZCoVzy4K+WpjulxBb1MZGKwBMloxhzBInW5UPopR9sQZclEnSgY\nCh6jWhuyzikR3kkC9rxc09CmzWrmbU/N+NULfoNd/KZXO3E/uqg7DEs9CjWHIghN\nPB/qjVEhAoGBAPyFsjbgRh3Zv7jeDQ67CPnPWh/q7rEITZCKRagJ3ACOF+ijTkWo\n+dAeqs0PoW+4CnNqx7HAxHgQ4SeZPbv9oyaXZUd3jzOpxCuV+De6igohP5qrjR0i\nmj59nDdT4T9inzAs1ElRXQssG8qevzzHB5MmOZM2mItOHH++0Ud7Wt+XAoGBANmA\n2YsX/wN6yufkSR0IGvLLfx7/adodtDZ1n7UGOo8udH9aKjBzl47Hepc/RjRlvS6V\nJlzvOMZNH0Z0yZbLvMuQkdFyglhYeLx/SmUL7ybBlb7/QlCBiGaZfiaaZBLgHWr4\n8V5iKlzj1Ao+8Mn6tSj8YHSqI8mOtQZpsS0ycDC5AoGAA5lMLugHV8mQp+vSN9GG\nkTjZSfcpK7C4mkS+NWTek8tyn8gkB24fEU4+lOmSHWt8CqUM74WVxzhGXTAb5x/4\nQUaLFPepPM1AlHZwsSqhaP+MToH/YtjpZdaYcVlqrmKTbjZVWC4mq1AXnU2h4BXe\nD8TNsUFn7yRP16o6hVBGvUUCgYAqMe8CJvOYDzhR6F2uviXMOGI+9znn0J9neUY0\nbjLqGA8NrcZFhAdA8b38nY/XFm2vHcxFdztCbS/GEV4SXRARRcikI1zaGr/BgchC\n9h+9Gw0b8pVA3QBDNz/b6VPEvam3WPgqYUzqnGBEZJV9+Z8vhlaIC4HJ1l+UEOkI\nZaKSMQKBgEZKzgoNlFMlfUR0nzCBpEdn4BOKLYfYYurVJw6lcx9hE9lEnRHBv/zO\nb4iNjA8pqLFfoi/O2luUG4lutDhGWYkEYnc4hPjityJFFtimw920CIFg+HCMOjoz\n+pLVg/dMnNUJpA8Hr7Kqh+Qsewo6s2HDwc0ZmanJp+99Lr4yn1OL\n-----END RSA PRIVATE KEY-----\n",
            "IssuingCA": "-----BEGIN CERTIFICATE-----\nMIICqDCCAZACCQCd1SFT0vZFKDANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDDApm\nbG9tZXNoLmlvMCAXDTIzMDgwNzA4NTA0NVoYDzMwMjIxMjA4MDg1MDQ1WjAVMRMw\nEQYDVQQDDApmbG9tZXNoLmlvMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC\nAQEAwMQdfgcRZxsoTTqmSU2PTGrLP7aqwteoMw0IULtvb17Fa5k3bVQj6Ob7ztUD\n77/SMqq+VXG4vE+TBx26pheiKRrClfvlPl+eS7jnTJQkQkuhsuJHmTfM/+mwcw7N\nuAIatKeq6CRaTnMAbfPqa7Baaotr/md1ThGImPj3E294MVbi8MIME3XlbFuB6YfK\nebTIcZOF4WzDskL8TwMqYZS3UNeZA4YALd68UQQHymIA6G97tuRh62kI3dZx1+RR\nJ1JP9gSFp1sbDD1mVY732w9olkv5K1KvKgDb+Ct8aqrG2Xmtrst7k37bQsszZcCb\ndDRS+qwYQSOT0iQAli4ITGH9tQIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQBKx4Jd\nVcJvnls2tDozUvAjdmN/RfWv9kurUJX6G+lR+2YoZF4BPEDraRiDeWjmV9NATofg\nk7AbHYSBDN1QQXsCmOtQzU4pTblNxglpAlOWbhw1BD1CkFAb6kqV9NnC3AJPPril\nwXR/Mznk180SXkXo0uQbvlMgrSTZkp0gzIvUSCMEFuzeONqPlT1GCNFeEb8W1JKM\n7eCs2j1tY/1LDTDJg2YB2nZHLMhNwHjQ6mef5QGu8eqK5Ijpzou3FrVZ9no1isQB\nwARkMU3HXVeSipUxr8/uS4ZmJ1jPCrLYBrxUiDYeGrfZAN0ZGMu1s9Z9VX+QBowq\nimKzAYRDka7IzVMh\n-----END CERTIFICATE-----\n"
          }
        ]
      }
    }
  ],
  "RouteRules": {
    "8080,8443": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": 100
            }
          }
        ]
      }
    }
  },
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  },
  "Chains": {
    "HTTPRoute": [
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ],
    "HTTPSRoute": [
      "common/tls-termination.js",
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

执行下面的命令发送请求进行测试。

```json
curl --cacert ca-cert.pem https://example.com --connect-to example.com:443:127.0.0.1:8443
```
