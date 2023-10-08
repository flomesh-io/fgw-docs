---
title: "会话保持"
description: "会话保持功能允许用户的连续请求在一段时间内被定向到同一台后端服务器。本文档将介绍如何使用 FGW 的会话保持功能。"
weight: 15
---

## 介绍

会话保持功能允许用户的连续请求在一段时间内被定向到同一台后端服务器。它通常在需要连续交互或状态跟踪的场景中被使用，例如在线购物车、用户登录状态或多步骤的事务处理。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 用于会话保持的两个服务，可以使用 Pipy 快速模拟。

```shell
#svr-1
pipy -e "pipy().listen(8081).serveHTTP(new Message({status: 200},'Hello, world'))"
#svr-2
pipy -e "pipy().listen(8082).serveHTTP(new Message({status: 503},'Service unavailable'))"
```

两个后端服务返回不同的状态码，用于区分会话保持是否生效。

```shell
curl -i http://localhost:8081
HTTP/1.1 200 OK
content-length: 12
connection: keep-alive

Hello, world

curl -i http://localhost:8082
HTTP/1.1 503 Service Unavailable
content-length: 19
connection: keep-alive

Service unavailable
```

## 配置说明

会话保持属于服务粒度的策略配置，可以参考 [服务配置文档](/reference/configuration/#41-服务-配置格式)。

- `StickyCookieName`：使用 cookie 保持会话的负载均衡时的 cookie 名称。这个字段是非必须的，但当启用基于 cookie 的会话保持时，它定义了存储后端服务器信息的 cookie 的名称，例如 `_srv_id`。这意味着当用户首次访问应用时，一个名为 `_srv_id` 的 cookie 会被设置，其值通常对应于某个后端服务器。当该用户再次访问时，此 cookie 会确保他们的请求被路由到与之前相同的服务器。
- `StickyCookieExpires`：使用 cookie 保持会话时，cookie 的有效期。这定义了 cookie 的存活时间，即多长时间内用户的连续请求会被定向到同一台后端服务器。参考值为 3600，意味着该 cookie 在一小时后过期。在此期间，所有来自同一用户的请求都会被路由到他们首次访问时的那台服务器。一旦 cookie 过期，下一次的请求可能会被路由到任何后端服务器，同时会收到一个新的 cookie。

### 示例

```json
{
  "Services": {
    "backendService1": {
      "StickyCookieName": "_srv_id",
      "StickyCookieExpires": 3600,
      "Endpoints": {}
    }
  }
}
```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081` 和 `127.0.0.1:8082`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，重试的功能在 `http/forward.js"` 插件中实现。

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

使用同一个 cookie 多次请求会被负载均衡到不同的后端服务。

```shell
curl -b "_srv_id=client-1" localhost:8080
Service unavailable
curl -b "_srv_id=client-1" localhost:8080
Hello, world
curl -b "_srv_id=client-1" localhost:8080
Service unavailable
```

### 会话保持配置

添加会话保持的配置，使用 `_srv_id` 作为 cookie 名。

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
      },
      "StickyCookieName": "_srv_id",
      "StickyCookieExpires":  3600
    }
  }
}

```

配置生效后，通过负载均衡器访问后端服务，负载均衡器会在响应的头部添加 cookie。

```shell
curl -i localhost:8080
HTTP/1.1 200 OK
set-cookie: _srv_id=7376698948304402; path=/; expires=Mon, 14 Aug 2023 16:14:33 GMT; max-age=3600
content-length: 12
connection: keep-alive

Hello, world
```

再次访问，但这次带上前面返回的 cookie。可以看到，每次都能获取到同样的结果，说明同一个会话（cookie 值相同）下，所有的请求都由相同的后端端点处理。

```shell
curl -b "_srv_id=7376698948304402" localhost:8080
Hello, world
curl -b "_srv_id=7376698948304402" localhost:8080
Hello, world
curl -b "_srv_id=7376698948304402" localhost:8080
Hello, world
```
