---
title: "故障注入"
description: "本篇文档将介绍如何在网关层面注入特定的故障来测试系统的行为和稳定性。"
weight: 13
draft: false
---

## 介绍

网关的故障注入功能允许在网络层面引入特定故障，如延迟和中断，以测试和评估应用在不同故障情况下的行为和稳定性。这个功能特别适合进行弹性和容错能力的测试。

这个功能特别适合进行弹性和容错能力的测试。

### 功能特点

- 延迟注入：模拟网络延迟，观察应用对延迟的响应。
- 错误注入：模拟服务错误响应，测试系统的恢复和容错机制。
- 可配置的故障比例：可以设定故障注入的覆盖范围，例如只对一定比例的请求注入故障。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

后端服务可以使用 Pipy 来模拟一个返回正在请求的路径的服务。

```shell
pipy -e "pipy().listen(8081).serveHTTP(msg => new Message('You are requesting ' + msg.head.headers.host + msg.head.path))"
```

它会返回我们正在请求的路径。

```shell
curl http://localhost:8081/user
You are requesting localhost:8081/user
```

## 配置说明

FGW 提供了三种粒度的故障注入：[网关全局](/reference/configuration/#1-全局配置configs)、[域名](/reference/configuration/#311-域名)和[路由](/reference/configuration/#3111-matches)。

`Fault`: 用于配置故障注入策略。
- `Delay`: 配置响应延时的参数。
  - `Percent`: 对请求注入延时的百分比，决定了有多少比例的请求会被注入延时。
  - `Fixed`: 指定固定的延时时间（如果设置），单位由 `Unit` 参数确定。
  - `Range`: 提供延时时间的随机范围，如 "0-100"，表示延时时间在0到100毫秒之间随机变动。
  - `Unit`: 延时时间的单位，支持 “ms”、“s”、“m”	，默认为毫秒（ms）。
- `Abort`: 配置故障响应码的参数。
  - `Percent`: 注入特定错误响应的请求比例。
  - `Status`: 指定要注入的HTTP或GRPC错误状态码，如503表示服务不可用。
  - `Message`: 可选，提供错误响应的提示信息。

### 示例

在这个配置示例中，30% 的请求将遇到 50 至 300 毫秒之间的随机延迟，5% 的请求将收到带有 “Error injected” 消息的 503 错误响应。

```json
{
  "Fault": {
    "Delay": {
      "Percent": 30,
      "Fixed": 2000,
      "Range": "50-300",
      "Unit": "ms"
    },
    "Abort": {
      "Percent": 5,
      "Status": 503,
      "Message": "Error injected"
    }
  }
}
```

## 配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的端点列表为我们的后端服务地址 `127.0.0.1:8081`。然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `http/fault-injection.js` 插件配置在相应的配置。

```json
{
  "Configs": {
    "DefaultPassthroughUpstreamPort": 443,
    "EnableDebug": true
  },
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
      "http/fault-injection.js",
      "http/service.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

### 路由粒度的故障注入

为 `/user` 路径增加一条路由，并在该路由上设置故障注入：`50%` 的概率下返回 `503` 的状态码，以及 `Error injected` 响应。

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
              "Path": "/user"
            },
            "BackendService": {
              "backendService1": {
                "Weight": 100
              }
            },
            "Fault": {
              "Abort": {
                "Percent": 50,
                "Status": 503,
                "Message": "Error injected"
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
  }
}
```

多次请求 `/user` 50% 的情况下会遇到 503 的错误，而请求其他路径正常返回。

```shell
curl -i localhost:8080/user

HTTP/1.1 503 Service Unavailable
content-length: 14
connection: keep-alive

Error injected

curl -i localhost:8080/user

HTTP/1.1 200 OK
content-length: 38
connection: keep-alive

You are requesting 127.0.0.1:8081/user
```

### 域名粒度的故障注入

**删除路由上的故障注入配置。**

接下来，我们在域名 `*` 上配置故障注入：`100%` 的情况下加入 `2000ms` 的延迟。

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
              "backendService1": {
                "Weight": 100
              }
            }
          }
        ],
        "Fault": {
          "Delay": {
            "Percent": 100,
            "Fixed": 2000,
            "Unit": "ms"
          }
        }
      }
    }
  }
}
```

此时测试可以发现响应的耗时都在 2s 以上。

```shell
time curl localhost:8080/user
You are requesting 127.0.0.1:8081/usercurl localhost:8080/user  0.00s user 0.01s system 0% cpu 2.032 total
```

### 网关粒度的全局配置

**删除域名上的故障注入配置。**

添加全局配置，设置 `100%` 的情况下加入 `1000ms` 的延迟以及返回 `401 Unauthorized!` 响应。

```json
{
  "Configs": {
    "DefaultPassthroughUpstreamPort": 443,
    "EnableDebug": true,
    "Fault": {
      "Delay": {
        "Percent": 100,
        "Fixed": 1000,
        "Unit": "ms"
      },
      "Abort": {
        "Percent": 100,
        "Status": 401,
        "Message": "Unauthorized!\n"
      }
    }
  }
}
```

测试。

```shell
time curl -i localhost:8080/user
HTTP/1.1 401 Unauthorized
content-length: 14
connection: keep-alive

Unauthorized!
curl -i localhost:8080/user  0.01s user 0.01s system 1% cpu 1.028 total
```