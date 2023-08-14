---
title: "路由"
description: "在本文档中我们将介绍 FGW 的路由功能"
weight: 0
---

## 介绍

在 L7（应用层）负载均衡中，FGW 能够根据 HTTP/HTTPS 的请求内容（例如 URL，头部信息等）来决定如何将流量路由到后端服务。

通过高度灵活的路由规则配置，FGW 能够满足各种复杂的路由需求。

在介绍 [HTTP/HTTPS 负载均衡](/features/http-load-balancer/) 时，我们将两个后端服务作为负载均衡的两个端点，对外提供服务。在本篇文档中，我们将分别为两个后端服务配置独立的路由。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务

## 配置

1. 第一步仍然是从配置端口监听开始，这里我们使用 `HTTP` 协议的端口。参考文档[监听端口配置](/reference/configuration/#2-监听端口配置listeners)。

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

2. 假设两个服务的路由分别是 `/hello` 和 `/hi`，参考路由的配置文档 [HTTP 协议端口号路由规则配置](/reference/configuration/#31-端口号配置protocol-为-httphttps-的配置格式)。后端服务这里，我们使用 `backendService1` 和 `backendService2` 分别代表两个后端服务。

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
              "Path": "/hello"
            },
            "BackendService": {
              "backendService1": 100
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/hi"
            },
            "BackendService": {
              "backendService2": 100
            }
          }
        ]
      }
    }
  }
}
```

3. 后端服务的配置，可以参考文档[服务配置](/reference/configuration/#4服务配置services)。

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

4. 最后是插件链的配置，仍然使用最小功能集的插件链。完成的 HTTP 插件链配置可以参考[文档](/reference/plugin/#http-路由)。 

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

5. 最终的配置如下。

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
              "Path": "/hello"
            },
            "BackendService": {
              "backendService1": 100
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/hi"
            },
            "BackendService": {
              "backendService2": 100
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
    },
    "backendService2": {
      "Endpoints": {
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