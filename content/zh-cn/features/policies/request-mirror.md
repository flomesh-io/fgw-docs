---
title: "流量镜像"
description: "本文档将介绍如何在不影响生产流量的前提下，将网络流量的副本发送到另一个服务。"
weight: 12
draft: false
---

## 介绍

FGW 的流量镜像功能主要用于在不影响生产流量的前提下，将网络流量的副本发送到另一个服务。这项功能常用于故障排查、性能监控、数据分析和安全审计等场景。通过流量镜像，可以实现实时数据捕获和分析，而不会对现有的业务流程造成任何干扰。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 2 个后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

使用 Pipy 模拟两个后端服务，这两个服务分别监听 `8081` 和 `8082` 端口。

```shell
pipy -e "pipy().listen(8081).serveHTTP(msg=>(console.log(msg.head),msg))"
pipy -e "pipy().listen(8082).serveHTTP(msg=>(console.log(msg.head),msg))"
```

当请求后端服务时，其会将请求的头部信息打印出来。

```shell
#request
curl localhost:8081/query
#log
2023-11-29 15:42:58.803 [INF] { protocol: "HTTP/1.1", headers: { "host": "localhost:8081", "user-agent": "curl/8.1.2", "accept": "*/*" }, method: "GET", scheme: undefined, authority: undefined, path: "/query" }
```

## 配置说明

如同在 [策略概览](/features/policies/) 中提到的，流量镜像策略可以应用在服务/路由粒度。它的配置只有一个字段：

- `BackendService`：指定请求应该镜像到哪个后端服务。

### 示例

```json
{
  "Filters": [
    {
      "Type": "RequestMirror",
      "BackendService": "backendService2"
    }
  ]
}
```

## 配置

### 基础配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的端点列表为我们的后端服务地址 `127.0.0.1:8081`。然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `filter/request-mirror.js` 插件配置在相应的配置。

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
      "filter/request-mirror.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

通过 FGW 访问后端服务 `curl localhost:8080/query`，可以看到 `8081` 的后端服务会打印请求的头部信息。

```shell
2023-11-29 16:15:15.306 [INF] { protocol: "HTTP/1.1", headers: { "host": "127.0.0.1:8081", "user-agent": "curl/8.1.2", "accept": "*/*", "x-forwarded-for": "127.0.0.1" }, method: "GET", scheme: undefined, authority: undefined, path: "/query" }
```

### 路由粒度的流量镜像

修改路由的配置，在路由上添加 `RequestMirror` 配置，将流量镜像到后端服务 `backendService2`。

```json
{
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
      },
      "Filters": [
        {
          "Type": "RequestMirror",
          "BackendService": "backendService2"
        }
      ]
    }
  ]
}
```

同时，配置后端服务 `backendService2` 的端点。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        }
      }
    },
    "backendService2": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      }
    }
  }
}
```

此时再次发送请求，会发现后端服务 `8082` 也会同样打印出请求的信息。

```
2023-11-29 16:48:28.153 [INF] { protocol: "HTTP/1.1", headers: { "host": "127.0.0.1:8082", "user-agent": "curl/8.1.2", "accept": "*/*", "x-forwarded-for": "127.0.0.1" }, method: "GET", scheme: undefined, authority: undefined, path: "/query" }
```

### 服务粒度的流量镜像

在前面的配置基础上调整：将路由负载均衡到后端服务 `8081` 和 `8082`，但是在选择 `8081` 作为后端服务时，将请求镜像到 `8082`。

```json
{
  "Matches": [
    {
      "Path": {
        "Type": "Prefix",
        "Path": "/"
      },
      "BackendService": {
        "backendService1": {
          "Weight": 100,
          "Filters": [
            {
              "Type": "RequestMirror",
              "BackendService": "backendService2"
            }
          ]
        },
        "backendService2": {
          "Weight": 100
        }
      }
    }
  ]
}

在测试的时候，还是同样的请求会被均衡到两个后端服务，但 `8081` 的请求会被镜像到 `8082`，也就是说 `8082` 将会收到所有的请求。