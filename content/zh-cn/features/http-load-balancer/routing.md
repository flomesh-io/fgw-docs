---
title: "路由"
description: "在本文档中我们将介绍 FGW 的路由功能"
weight: 0
---

## 介绍

路由在计算机网络中是一个至关重要的概念。FGW 的路由功能是专为现代 Web 应用和微服务设计的。它提供了细粒度的控制，让开发者能够精确地控制网络流量的流向和行为。这在现代的微服务架构中尤为关键，因为它可以优化网络性能、增强安全性，并提供更好的用户体验。常见的使用场景有：

1. **蓝绿部署和灰度发布**: 通过 FGW 的路由功能，开发者可以轻松地将流量定向到新版本的服务，进行 A/B 测试，或逐步部署新功能，确保平稳过渡。
2. **API 版本管理**: 通过不同的路由规则，您可以管理多版本的 API，并在不中断服务的情况下进行迁移。
3. **安全性增强**: 通过具体的 HTTP 头部或请求参数匹配，FGW 可以有效地拦截恶意请求或重定向到安全审核流程。

在 L7（应用层）负载均衡中，FGW 能够根据 HTTP/HTTPS 的请求内容（例如 URL，头部信息等）来决定如何将流量路由到后端服务。

通过高度灵活的路由规则配置，FGW 能够满足各种复杂的路由需求。更多的高级功能，请参考 [FGW 策略](/features/policies/) 部分。

在介绍 [HTTP/HTTPS 负载均衡](/features/http-load-balancer/) 时，我们将两个后端服务作为负载均衡的两个端点，对外提供服务。在本篇文档中，我们将分别为两个后端服务配置独立的路由。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务

## 配置说明

路由规则 (`RouteRules`) 的配置:

- `端口号`: 定义哪些端口将应用特定的路由规则。这使得微服务或应用能够在不同的端口上根据需求使用不同的路由策略。例如：`80` 或 `80, 443`。
    - `域名`: 用于识别虚拟主机并根据请求的目标域名决定如何路由流量。例如：`*` 或 `www.test.com, api.test.com`。
        - `RouteType`: 描述了此规则的目标流量类型，决定了数据应如何被解析和路由。例如：`HTTP`、`GRPC`。
        - `Matches`: 为请求提供具体的匹配条件，只有满足这些条件的请求才会被路由。
            - `Path`: 对请求的 URI 路径进行匹配，从而决定相应的路由策略。
                - `Type`: 描述了如何匹配路径，是否是精确匹配、前缀匹配还是正则表达式匹配。例如：`Prefix`、`Exact`、`Regex`。
                - `Path`: 真实要匹配的路径值。例如：`/`, `/prefix`。
            - `Headers`: 根据 HTTP 请求头部的特定值决定如何路由流量。
                - `Exact`: 针对头部值进行精确匹配。
                - `Regex`: 允许复杂的正则表达式模式匹配，提供更大的灵活性。
            - `Methods`: 限制只有特定的 HTTP 方法的请求才能被路由，提高安全性和效率。例如：`GET`、`POST`、`DELETE`、`PUT`、`HEAD`、`OPTION` 等。
            - `QueryParams`: 对请求参数进行匹配，为 API 版本控制或特性开关提供能力。
                - `Exact`: 对查询参数进行精确匹配。
                - `Regex`: 允许对查询参数使用正则匹配。
            - `BackendService`: 指定请求应该路由到哪个后端服务，使得多个微服务可以平滑地共享一个入口点。
            - `ServerRoot`: 为那些需要提供静态内容的服务提供一个物理路径，如网站的首页或资源。
            - `Method`: 当路由的流量类型为 `GRPC` 时，这是决定如何处理流量的关键组件。
                - `Type`: 描述如何匹配 GRPC 请求。例如：`Exact`。
                - `Service`: 确定请求是针对哪个 GRPC 服务。例如：`com.example.GreetingService`。
                - `Method`: 确定请求是针对服务中的哪个具体方法。例如：`Hello`。
            - `RateLimit`: 根据预设的标准对流量进行限制，确保服务不会被过多的请求淹没。属于路由级别的限流，参考 [限流策略](/features/policies/rate-limiting/) 的使用。
        - `RateLimit`: 在域名级别对流量进行限制，为每个域名提供特定的限制标准。参考 [限流策略](/features/policies/rate-limiting/) 的使用。

### 示例

```json
{
  "RouteRules": {
    "80": {
      "test.com": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/path"
            },
            "Headers": {
              "Exact": {
                "a": "1",
                "b": "2",
                "c": "3"
              }
            },
            "Methods": [
              "GET",
              "POST"
            ],
            "QueryParams": {
              "Exact": {
                "abc": "1"
              }
            },
            "BackendService": {
              "www8088": 100
            }
          }
        ]
      },
      "*.test.com": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/repo"
            },
            "Headers": {
              "Exact": {
                "b": "2"
              }
            },
            "Methods": [
              "GET",
              "POST"
            ],
            "QueryParams": {
              "Exact": {
                "abc": "1"
              }
            },
            "BackendService": {
              "www8088": 100
            }
          }
        ]
      }
    }
  }
}
```

## 配置

1. 第一步仍然是从配置端口监听开始，这里我们使用 `HTTP` 协议的端口。参考文档 [监听端口配置](/reference/configuration/#2-监听端口配置listeners)。

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
              "backendService1": {
                "Weight": 100
              }
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/hi"
            },
            "BackendService": {
              "backendService2": {
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

3. 后端服务的配置，可以参考文档 [服务配置](/reference/configuration/#4-服务配置services)。

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

4. 最后是插件链的配置，仍然使用最小功能集的插件链。完成的 HTTP 插件链配置可以参考 [文档](/reference/plugin/#http-路由)。

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
              "backendService1": {
                "Weight": 100
              }
            }
          },
          {
            "Path": {
              "Type": "Prefix",
              "Path": "/hi"
            },
            "BackendService": {
              "backendService2": {
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
