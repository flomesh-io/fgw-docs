---
title: "黑白名单"
description: "本文档介绍如何使用 FGW 的黑白名单功能对访问来源进行控制"
weight: 17
---

## 介绍

在微服务和 API 网关中，黑白名单功能是一种常用的安全机制，它可以根据指定的 IP 地址或 IP 地址范围来控制对服务或路由的访问。通过 FGW 的黑白名单功能，可以轻松地对 IP 进行访问控制，提高服务的安全性。

使用黑白名单，可以精确地控制哪些 IP 可以访问的服务，以及哪些 IP 应被禁止。这种功能在对外公开的 API 或面向特定客户或合作伙伴的服务中特别有用。

- **黑名单**：一旦 IP 地址被列入黑名单，来自这些 IP 地址的请求将被 FGW 拒绝。这通常用于封锁可疑的流量或已知的恶意 IP 地址。
- **白名单**：仅允许列在白名单中的 IP 地址访问。不在白名单上的所有 IP 都将被 FGW 拒绝。这通常用于确保只有特定的合作伙伴或内部 IP 可以访问特定的服务或路由。

黑白名单功能支持多种粒度的控制：端口、域名（服务）以及路由，通过在不同级别配置黑白名单来可以完成细粒度地灵活控制，可根据使用场景灵活选择。

- 端口级别的控制在建立连接时就会进行检查，不被允许的 IP 会直接拒绝连接，效率会更好。
- 服务和路由级别的控制粒度更小更灵活，但是需要在 HTTP 协议解码之后才可以进行检查，相比端口级别的黑白名单效率相对较低。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

这里的后端服务，可以使用 Docker 运行一个 httpbin 服务。

```shell
docker run --rm -d --name httpbin -p 8081:80 kennethreitz/httpbin
```

这个服务提供多个端点，比如 `/ip` 和 `/headers`。

```shell
curl -i http://localhost:8081/status/200
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Thu, 31 Aug 2023 03:22:37 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Content-Length: 0

curl http://localhost:8081/headers
{
  "headers": {
    "Accept": "*/*",
    "Host": "localhost:8081",
    "User-Agent": "curl/8.1.2"
  }
}
```

## 配置说明

访问控制列表（`AccessControlLists`）的配置字段：

- `blacklist`：列出要阻止访问的 IP 地址或子网。例如, `["1.1.1.1", "2.2.2.0/24"]`。如果指定了该配置，FGW 将拒绝列出的 IP 地址或范围内的所有请求。
- `whitelist`：列出允许访问的 IP 地址或子网。例如, `["1.1.1.1", "2.2.2.0/24"]`。如果指定了该配置，FGW 仅允许列出的 IP 地址或范围内的请求访问。
- `enableXFF`：是否检查请求头部 `x-forwarded-for` 中的 IP 地址（逗号分隔），默认不检查。
- `status`：拒绝访问后返回的响应状态码，不指定时默认为 403，可以用于定制响应。
- `message`：拒绝访问后返回的响应内容，不指定时默认为空字符，可以用于定制响应。

## 示例配置

```json
{
  "AccessControlLists": {
    "enableXFF": true,
    "status": "403",
    "message": "IP Forbidden",
    "blacklist": [
      "1.1.1.1",
      "2.2.2.0/24"
    ],
    "whitelist": [
      "3.3.3.3",
      "4.4.4.0/24"
    ]
  }
}
```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](app://obsidian.md/features/http-load-balancer/) 中的负载均衡器配置，修改服务的端点列表为我们的后端服务地址 `127.0.0.1:8081`；然后根据 [HTTP 插件链配置](app://obsidian.md/reference/plugin/#http-%E8%B7%AF%E7%94%B1)，将 `common/access-control.js`、`http/access-control-domain.js`、`http/access-control-route.js` 插件配置在相应的配置。

这里我们会多加入一个监听端口 `8079`，与端口 `8080` 使用同样的负载均衡配置。当我们做端口级别的黑白名单时，方便进行进行对比验证。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8079
    },
    {
      "Protocol": "HTTP",
      "Port": 8080
    }
  ],
  "RouteRules": {
    "8079,8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/ip"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
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
        }
      }
    }
  },
  "Chains": {
    "HTTPRoute": [
      "common/access-control.js",
      "common/consumer.js",
      "http/codec.js",
      "http/route.js",
      "http/service.js",
      "http/access-control-domain.js",
      "http/access-control-route.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

通过负载均衡访问后端服务。

```shell
curl http://localhost:8080/ip
{
  "origin": "192.168.215.1"
}

curl http://localhost:8080/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "localhost:8080",
    "User-Agent": "curl/8.1.2"
  }
}

curl http://localhost:8079/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "localhost:8079",
    "User-Agent": "curl/8.1.2"
  }
}
```

### 端口级别黑白名单配置

我们在监听端口 `8080` 上配置黑名单，因为请求发起方和代理都位于同一机器上，IP 地址使用 `127.0.0.1`。

```json
{
  "Listeners": [
    {
      "Protocol": "HTTP",
      "Port": 8080,
      "AccessControlLists": {
        "blacklist": [
          "127.0.0.1"
        ]
      }
    }
  ]
}
```

我们发起请求验证是否生效。

```shell
curl http://localhost:8080/headers
curl: (56) Recv failure: Connection reset by peer

curl http://localhost:8079/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "localhost:8079",
    "User-Agent": "curl/8.1.2"
  }
}
```

### 域名级别黑白名单设置

**记得移除上面配置的端口级别黑白名单配置。**

接下来，我们将同样的配置挂在域名 `*` 下。

```json
{
  "RouteRules": {
    "8079,8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/ip"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            }
          }
        ],
        "AccessControlLists": {
          "blacklist": [
            "127.0.0.1"
          ]
        }
      }
    }
  }
}
```

此时，访问 `8080` 和 `8079` 都会收到 `403` 的响应。

```shell
curl -i http://localhost:8080/headers
HTTP/1.1 403 Forbidden
content-length: 0
connection: keep-alive

curl -i http://localhost:8079/headers
HTTP/1.1 403 Forbidden
content-length: 0
connection: keep-alive
```

### 路由级别黑白名单设置

**记得移除上面配置的端口级别黑白名单配置。**

有些时候，我们希望更细粒度的控制，比如限制某些 IP 对某个路由的访问。我们将同样的配置挂在 `/ip` 路由下。

```json
{
  "RouteRules": {
    "8079,8080": {
      "*": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/ip"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            },
            "AccessControlLists": {
              "blacklist": [
                "127.0.0.1"
              ]
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            }
          }
        ]
      }
    }
  }
}
```

此时，我们访问 `/ip` 路由被限制，而其他的路由如 `/headers` 则无限制。

```shell
curl http://localhost:8080/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "keep-alive",
    "Host": "localhost:8080",
    "User-Agent": "curl/8.1.2"
  }
}

curl -i http://localhost:8080/ip
HTTP/1.1 403 Forbidden
content-length: 0
connection: keep-alive

curl -i http://localhost:8079/ip
HTTP/1.1 403 Forbidden
content-length: 0
connection: keep-alive
```
