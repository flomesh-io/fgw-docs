---
title: "限流"
description: "限流可以确保服务在高流量下仍然稳定，预防系统雪崩。本文档将介绍如何使用 FGW 的限流功能。"
weight: 2
---

## 介绍

限流（Rate Limiting）是一个非常重要的功能，尤其在现代的分布式应用程序中。

限流是控制数据请求速率的方法，以确保在任何给定的时间内，传入的请求数量不会超过系统预定义的限额。通过这种方式，限流可以保护系统避免因过多的请求而过载，从而维护服务的质量和响应速度。

FGW 中限流的粒度与 [熔断](/features/policies/circuit-breaking/) 等策略的服务维度少有不同，可以作用在域名和路由两个维度上，提供更加精细化的限流配置。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 这里后端服务建议使用工具 [Fortio](https://fortio.org)。Fortio 作为服务器运行时，可以在请求时指定延迟、响应状态，及其分布，非常方便测试熔断场景。同时 Forito 还可以作为负载生成器来使用。
>
> 启动服务器可以使用命令 `fortio server -http-port 8082`。

## 配置说明

### 限流配置

下面列出了限流配置的字段，也可以参考文档 [Ratelimit 配置](/reference/configuration/#3112-ratelimit)。

- `Backlog`: 积压值是指在达到限流阈值时，系统允许排队的请求数量。这是一个重要的字段，尤其是在系统突然遭受大量请求时，这些请求可能会超过设置的限流阈值。积压值为系统提供了一定的缓冲，可以处理超出限流阈值的请求，但在积压值上限内。一旦达到积压上限，任何新的请求都会立即被拒绝，无需等待。可选字段。
- `Requests`: 请求值是指在限流时间窗口内允许的访问次数。这是限流策略的核心参数，它确定了在特定的时间窗口内可以接受多少请求。设置此值的目的是为了确保在给定的时间窗口内，后端系统不会受到超过它能处理的请求。必须字段。
- `Burst`: 爆发值表示在短时间内允许的最大请求次数。这是一个可选字段，它主要用于处理短时间的请求高峰。爆发值通常大于请求值，允许在短时间内接受的请求数量超过平均速率。非必须字段。
- `StatTimeWindow`: 限流时间窗口（单位：秒）定义了统计请求数量的时间段。限流策略通常基于滑动窗口或固定窗口来实现。StatTimeWindow 定义了这个窗口的大小。比如，如果 `StatTimeWindow` 设置为 60 秒，并且 `Requests` 为 100，则意味着每 60 秒内最多只能有 100 个请求。必须字段。
- `ResponseStatusCode`: 发生限流时，返回给客户端的 HTTP 状态码。这个状态码告诉客户端请求被拒绝的原因是因为达到了限流阈值。常见的状态码是 `429（Too Many Requests）`，但可以根据需要自定义。必须字段。
- `ResponseHeadersToAdd`: 发生限流时，要添加到响应中的 HTTP 头部信息。这可以用来通知客户端有关限流策略的更多信息。例如，可以添加一个 `Retry-After` 头来告诉客户端多久后可以重试。还可以提供关于当前限流策略或如何联系系统管理员的其他有用信息。可选字段。

#### 示例

```json
{
  "RateLimit": {
    "Local": {
      "Backlog": 1,
      "Requests": 100,
      "Burst": 200,
      "StatTimeWindow": 60,
      "ResponseStatusCode": 429,
      "ResponseHeadersToAdd": [
        {
          "Name": "RateLimit-Limit",
          "Value": "100"
        }
      ]
    }
  }
}
```

### 限流的维度

**基于 Host 的限流**（参考 [域名配置文档](/reference/configuration/#311-域名)），可以所有对某个 Host 的请求进行统计限流。

```json
{
  "example.com": {
    "RouteType": "HTTP",
    "Matches": [],
    "RateLimit": {}
  }
}
```

**基于路由的限流**（参考路由 [配置文档](/reference/configuration/#3111-matches)），可以对匹配某个路由的请求进行统计限流。

```json
{
  "example.com": {
    "RouteType": "HTTP",
    "Matches": [
      {
        "Path": {},
        "BackendService": {},
        "RateLimit": {}
      }
    ]
  }
}
```

### 限流的作用范围

限流是为了控制到达系统或组件的请求速率。根据其作用范围和实施地点，限流策略可以分为**本地限流**和**全局限流**。

> **注：当前版本 {{< param fgw_version >}} 仅支持本地限流。**

#### 1. 本地限流 (Local Rate Limiting)

- **定义**：本地限流通常在单个实例或节点上实施，独立于其他节点。每个节点都根据自己的限流策略和规则来处理流入的请求。
- **作用范围**：只对单一实例或节点有效。
- **优势**：
    - **延迟较低**：由于限流判断在本地完成，无需与其他系统或节点通信，因此延迟很小。
    - **简单**：通常很容易实施和管理。
- **劣势**：
    - **非均匀限流**：由于每个节点都是独立的，因此在高流量下，某些节点可能会被快速饱和，而其他节点可能还有未使用的容量。
    - **缺乏集中的控制和可见性**：各个节点可能无法获知整体系统的状态。

#### 2. 全局限流 (Global Rate Limiting)

- **定义**：全局限流通常在系统或服务的所有实例或节点之间共享。所有的请求，不论它们到达哪个节点，都会计入全局的限流计数器。
- **作用范围**：涵盖了整个系统或服务的所有实例。
- **优势**：
    - **均匀限流**：确保整个系统都遵循相同的限流规则，避免了单个节点的过载。
    - **集中的控制和可见性**：提供一个集中的位置来配置、管理和监视限流，使得运维更为简便。
- **劣势**：
    - **潜在的延迟**：为了同步全局计数器，节点间可能需要进行通信，这可能会增加延迟。
    - **实施复杂性**：全局限流策略需要更多的协调和同步机制，可能需要额外的组件（如共享的数据存储或分布式计数器）。

#### 应用场景

- 对于小型或低流量的系统，**本地限流**可能就足够了，因为它简单且开销小。
- 对于大型、分布式或高流量的系统，**全局限流**可能更为适合，因为它可以更有效地控制和分配请求，尤其是在多个节点或实例间。

最终，选择本地限流还是全局限流取决于系统的规模、需求以及预期的流量模式。

## 配置

### 负载均衡器配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8082`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `http/throttle-domain.js` 和 `http/throttle-route.js` 插件配置在相应的配置。

顾名思义，这两个插件将分别处理基于 Host 的限流和基于路由的限流。

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
      "http/throttle-domain.js",
      "http/throttle-route.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

### 基于域名的限流配置

在上面的负载均衡器配置中，使用了通配符 `*` 来匹配所有的域名，我们也可以将基于域名的限流赋给它。

下面增加限流的配置，时间窗口为 `60s`，最大请求数为 `100`，爆发值设置为 `200`。同时在被限流的响应头部加上限流值的提醒。

```json
{
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
    "RateLimit": {
      "Local": {
        "Backlog": 1,
        "Requests": 100,
        "Burst": 200,
        "StatTimeWindow": 60,
        "ResponseStatusCode": 429,
        "ResponseHeadersToAdd": [
          {
            "Name": "RateLimit-Limit",
            "Value": "100"
          }
        ]
      }
    }
  }
}
```

使用 Fortio 作为负载生成器：以 10 个并发、200 的 QPS 发送 1000 个请求。

```shell
fortio load -quiet -c 10 -n 1000 -qps 200 http://localhost:8080/echo
# target 50% 0.00204893
# target 75% 0.00281346
# target 90% 0.00371774
# target 99% 0.00742857
# target 99.9% 3.00293

IP addresses distribution:
127.0.0.1:8080: 802
Code 200 : 200 (20.0 %)
Code 429 : 800 (80.0 %)
```

上面将限流的统计时间窗口设置为了 1 分钟，因此在执行完 fortio load 命令后，仍然会处在限流状态。此时，我们使用 `curl` 来请求，可以查看响

```shell
curl -i http://localhost:8080/echo
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 100
content-length: 0
connection: keep-alive
```

上面是基于 Host 的限流配置，如果将这段配置移到路由上也可以得到同样的结果。

### 基于路由的限流

我们在原有的路由基础上，添加一条到 `/echo` 的路由，并将原来域名限流的配置移到这条路由上。

```json
{
  "*": {
    "RouteType": "HTTP",
    "Matches": [
      {
        "Path": {
          "Type": "Prefix",
          "Path": "/echo"
        },
        "BackendService": {
          "backendService1": {
            "Weight": 100
          }
        },
        "RateLimit": {
          "Local": {
            "Backlog": 1,
            "Requests": 100,
            "Burst": 200,
            "StatTimeWindow": 60,
            "ResponseStatusCode": 429,
            "ResponseHeadersToAdd": [
              {
                "Name": "RateLimit-Limit",
                "Value": "100"
              }
            ]
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
```

还是以同样条件进行负载测试，可以得到同样的测试结果。但是我们马上分别请求 `/echo` 和 `/`，可以看到两种不同的结果。

```shell
curl -i http://localhost:8080/echo
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 100
content-length: 0
connection: keep-alive

curl -i http://localhost:8080/
HTTP/1.1 200 OK
date: Sun, 13 Aug 2023 11:34:36 GMT
content-length: 0
connection: keep-alive
```

只有访问设置了限流的路由的请求才会被限流，其他路由并不会收到影响。
