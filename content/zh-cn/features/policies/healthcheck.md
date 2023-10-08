---
title: "健康检查"
description: "FGW 的健康检查功能可用于提升系统的高可用性。本文档将介绍如何使用 FGW 的健康检查功能。"
weight: 5
---

## 介绍

FGW 的健康检查功能允许自动监控和验证后端服务的健康状况。这样，FGW 可以确保流量只被转发到健康的、能够正常处理请求的服务。此功能尤其在微服务或分布式系统中至关重要，它确保了高可用性和弹性。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 用于健康检查的两个服务，可以使用 Pipy 快速模拟。

```shell
#healthy
pipy -e "pipy().listen(8081).serveHTTP(new Message({status: 200},'Hello, world'))"
#unhealthy
pipy -e "pipy().listen(8082).serveHTTP(new Message({status: 500},''))"
```

## 配置说明

参考 [健康检查配置文档](/reference/configuration/#411-healthcheck)。

- `Interval`：健康检查间隔，表示系统对后端服务进行健康检查的时间间隔。此字段决定了多久检查服务一次，以确保它们的健康状况。例如，如果设置为 10 秒，系统将每 10 秒检查一次上游服务的健康状态。这有助于在服务变得不可用时快速检测并作出反应。不设置此字段意味着不进行主动健康检查。可选字段。
- `MaxFails`：最大失败次数，定义了在将上游服务标记为不可用之前，可以连续失败的健康检查次数。这是一个关键参数，因为它决定了系统在决定服务不健康之前的容忍度。例如，如果设置为 3，那么在第三次连续的健康检查失败后，该服务将被标记为不可用。必须字段。
- `FailTimeout`：失败超时，定义了一个上游服务在被标记为不健康后，将被暂时禁用的时间长度。这意味着，即使服务再次变得可用，它也会在这段时间内被代理视为不可用。此字段在未设置主动健康检查的情况下尤为重要，因为它决定了一个服务在多长时间内被认为是不健康的。如果已设置主动健康检查，此参数会被忽略。可选字段。
- `Uri`：健康检查 URI，用于 HTTP 健康检查的路径。当使用 HTTP 方法进行健康检查时，系统会向此 URI 发送请求以确定服务的健康状态。如果此字段未设置，则健康检查将使用 TCP 方法检查端口的健康状况，而不是使用 HTTP 方法。可选字段。
- `Matches`：匹配条件，用于确定 HTTP 健康检查的成功或失败。此字段可以包含多个条件，例如期望的 HTTP 状态码、响应体内容等。当健康检查的响应满足这些条件时，服务将被认为是健康的。否则，它将被认为是不健康的。这允许更细粒度的控制，确保服务不仅是可达的，而且是正常运行的。可选字段，但如果使用 HTTP 健康检查，建议设置。
  - `Type`：匹配类型。这个字段定义了你想在健康检查响应中匹配的具体部分。它有三个有效的值：
      - `status`：匹配 HTTP 响应的状态码列表，如 `[200,201,204]`。
      - `body`：匹配 HTTP 响应的主体内容。
      - `headers`：匹配 HTTP 响应的头部信息。此字段是必须的。
  - `Value`：期望的数据。此字段与 `Type` 字段一起使用，定义了预期的匹配值。例如，如果 `Type` 设置为 `status`，`Value` 可能会设置为 `200`，表示期望 HTTP 响应的状态码为 200。此字段是必须的。
  - `Name`：当 `Type` 为 `headers` 时使用。它定义了你想在 HTTP 响应头部中匹配的具体字段名。例如，如果你想检查 `Content-Type` 头部的值，你会设置 `Name` 为 `Content-Type`。此字段只在 `Type` 设置为 `headers` 时有效，其他情况下不应该设置。此字段是可选的。

### 示例

```json
{
  "HealthCheck": {
    "Interval": 10,
    "MaxFails": 3,
    "FailTimeout": 30,
    "Uri": "/",
    "Matches": [
      {
        "Type": "status",
        "Value": [
          200,
          201
        ]
      }
    ]
  }
}
```

## 配置

### 负载均衡器配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081` 和 `127.0.0.1:8082`；重试功能实现在 `common/health-check.js` 中，该插件是自动引入，无需再插件链中显式定义。

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

在配置健康检查，我们可以看一下负载均衡的结果，一半的请求下返回了 500。

```shell
curl --head http://localhost:8080/
HTTP/1.1 200 OK
content-length: 12
connection: keep-alive

curl --head http://localhost:8080/
HTTP/1.1 500 Internal Server Error
content-length: 0
connection: keep-alive
```

### 健康检查配置

接下来为服务 `backendService1` 添加健康检查的配置。为了便于快速查看效果，我们讲检查的间隔时间设置为 2 秒，失败次数 2 次之后就被认定为不健康的端点。同时，设置检查的依据为 `200` 响应码。

> 实际场景中，过于频繁的健康检查虽然可以快速隔离不健康的端点，但也势必会给后端服务造成压力。同样较低的最大失败数判定，也会快速隔离不健康的端点提升服务的可用性，但也会因为偶尔的网络抖动、服务突发负载过高等原因导致大量端点“被”下线，导致服务的不可用。

```shell
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
      "HealthCheck": {
        "Interval": 2,
        "MaxFails": 2,
        "FailTimeout": 30,
        "Uri": "/",
        "Matches": [
          {
            "Type": "status",
            "Value": [
              200,
              201
            ]
          }
        ]
      }
    }
  }
}
```

配置生效后，再次进行测试可以看到不健康的端点已经被隔离。

```shell
curl --head http://localhost:8080/
HTTP/1.1 200 OK
content-length: 12
connection: keep-alive

curl --head http://localhost:8080/
HTTP/1.1 200 OK
content-length: 12
connection: keep-alive
```
