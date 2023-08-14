---
title: "熔断"
description: "熔断策略的引入，使得系统具备更强的容错能力和更优化的资源利用。本文档将介绍如何使用 FGW 的熔断功能。"
weight: 1
---

## 介绍

熔断是一种防止系统过载并保证服务可用性的机制。当一个服务连续失败达到一定次数时，Flomesh Gateway（FGW）将“熔断”该服务，暂时停止向其发送请求。这可以防止故障的蔓延，并给出故障服务恢复所需的时间。

熔断策略的引入，使得系统具备更强的容错能力和更优化的资源利用。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 这里后端服务建议使用工具 [Fortio](https://fortio.org)。Fortio 作为服务器运行时，可以在请求时指定延迟、响应状态，及其分布，非常方便测试熔断场景。同时 Forito 还可以作为负载生成器来使用。
>
> 启动服务器可以使用命令 `fortio server -http-port 8082`。

## 配置说明

在 FGW 中熔断是 [服务 Service 维度的配置](/reference/configuration/#41-服务-配置格式)，可以针对服务的慢访问、错误进行熔断，并支持配置熔断后的响应。这两种熔断的触发条件很容易理解：

- **慢访问触发熔断**：当系统中某个服务的响应时间超过预定阈值时，熔断器会开启，以阻止对该服务的进一步访问。
- **错误触发熔断**：当系统中某个服务的错误率超过预定阈值时，熔断器会开启，以阻止对该服务的进一步访问，比如服务返回大量的 5xx 错误。

### 熔断配置项

- `MinRequestAmount`: 触发熔断必须达到的访问次数。为了避免基于小样本的误判，这一字段确保在短时间内的极少数错误或慢访问不会导致过早的熔断。只有在统计时间窗口内的请求达到此设定值，系统才会考虑熔断策略。必须字段。
- `StatTimeWindow`: 熔断判定统计窗口（单位：秒）。这定义了评估慢访问和错误请求的时间范围。在这个窗口内，系统会收集相关数据，以判断是否需要熔断。必须字段。
- `SlowTimeThreshold`: 慢访问时间阈值（单位：秒）。它定义了什么样的响应时间应该被认为是“慢”的。如果一个请求的响应时间超过了这个阈值，它就被标记为慢访问。非必须字段。
- `SlowAmountThreshold`: 在 `StatTimeWindow` 时间统计窗口内，触发熔断的慢访问次数。达到或超过这个阈值的慢访问次数会触发熔断。非必须字段。
- `SlowRatioThreshold`: 触发熔断的慢访问占比（0.00~1.00）。这与上面的次数阈值不同。如果慢访问的占比超过这个设置的阈值，则会触发熔断。非必须字段。
- `ErrorAmountThreshold`: 在 `StatTimeWindow` 时间统计窗口内，触发熔断的失败访问次数。错误的访问次数达到或超过此阈值时，会触发熔断。非必须字段。
- `ErrorRatioThreshold`: 触发熔断的失败访问占比（0.00~1.00）。与次数阈值不同，如果失败的请求占比超过这个阈值，则会触发熔断。非必须字段。
- `DegradedTimeWindow`: 熔断时间窗口（单位：秒）。当熔断被触发后，服务将进入一个“降级”状态，只返回错误或预设响应，不再处理实际的请求，持续这个时间窗口长度。之后，服务会尝试自我恢复，再次接受并处理请求。必须字段。
- `DegradedStatusCode`: 当服务被熔断时，返回给客户端的 HTTP 状态码。这是通知客户端因为某种原因服务当前不可用的标准方式。必须字段。
- `DegradedResponseContent`: 当发生熔断时，返回给客户端的提示信息。这可以为客户端提供更具体的错误信息或建议的后续操作。非必须字段。

### 示例

```javascript
{
  "CircuitBreaking": {
    "MinRequestAmount": 10,
    "StatTimeWindow": 60,
    "SlowTimeThreshold": 1.5,
    "SlowAmountThreshold": 5,
    "SlowRatioThreshold": 0.1,
    "ErrorAmountThreshold": 3,
    "ErrorRatioThreshold": 0.05,
    "DegradedTimeWindow": 300,
    "DegradedStatusCode": 503,
    "DegradedResponseContent": "Service is temporarily unavailable."
  }
}
```

## 配置

### 负载均衡器配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8082`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `http/circuit-breaker.js` 插件配置在相应的配置。

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
      "http/circuit-breaker.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

前面提到熔断的配置是服务维度的，接下来我们开始熔断的配置。

### 慢访问数量熔断

在配置熔断之前，检查下负载均衡器对慢访问的反应。将服务端的延迟设置为 600ms，概率为 30。在统计延迟分布的时候，特地设置了 `71` 这个百分比。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?delay\=600ms:30

# target 50% 0.0058
# target 71% 0.601213
# target 90% 0.608895
# target 95% 0.610917
# target 99% 0.612534

IP addresses distribution:
127.0.0.1:8080: 10
Code 200 : 100 (100.0 %)
```

设置服务 `backendService1` 在 10 秒的时间统计窗口内、慢访问（延迟大于 `500ms`）的数量达到 `20` 时进行熔断。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      },
      "CircuitBreaking": {
        "MinRequestAmount": 10,
        "StatTimeWindow": "60",
        "ErrorAmountThreshold": "20",
        "DegradedTimeWindow": 10,
        "DegradedStatusCode": 503,
        "DegradedResponseContent": "Service is temporarily unavailable."
      }
    }
  }
}
```

在配置慢访问熔断后，再进行测试就会发现有 30% 的响应状态码为 503，熔断生效。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?delay\=600ms:30

# target 50% 0.0117143
# target 71% 0.0175385
# target 90% 0.610336
# target 95% 0.614643
# target 99% 0.618088

IP addresses distribution:
127.0.0.1:8080: 30
Code 200 : 70 (70.0 %)
Code 503 : 30 (30.0 %)
```

### 慢访问占比熔断

同样，假如我们将慢访问熔断从**数量触发**熔断改成**百分比触发**， 将 `SlowAmountThreshold=20` 改为 `SlowRatioThreshold=0.2`，即慢访问的占比达到 20% 触发熔断。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      },
      "CircuitBreaking": {
        "MinRequestAmount": 10,
        "StatTimeWindow": "60",
        "SlowTimeThreshold": "0.5",
        "SlowRatioThreshold": 0.2,
        "DegradedTimeWindow": 10,
        "DegradedStatusCode": 503,
        "DegradedResponseContent": "Service is temporarily unavailable."
      }
    }
  }
}
```

还是用同样的方式进行测试，可以发现有 80% 的请求被熔断。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?delay\=600ms:30

# target 50% 0.00353846
# target 71% 0.00433333
# target 90% 0.00733333
# target 95% 0.601387
# target 99% 0.606937

IP addresses distribution:
127.0.0.1:8080: 80
Code 200 : 20 (20.0 %)
Code 503 : 80 (80.0 %)
```

假如设置 15% 的响应延迟 600ms 的话，并不会触发熔断。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?delay\=600ms:15

# target 50% 0.00286111
# target 71% 0.00469231
# target 90% 0.603466
# target 95% 0.606932
# target 99% 0.609705

IP addresses distribution:
127.0.0.1:8080: 10
Code 200 : 100 (100.0 %)
```

### 错误响应数触发熔断

我们先看下没有配置熔断的情况下，服务端 30% 响应返回 503 的情况（实际数据会有一定的浮动 +- 5%）。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?status\=500:30

IP addresses distribution:
127.0.0.1:8080: 37
Code 200 : 68 (68.0 %)
Code 500 : 32 (32.0 %)
```

我们设置触发熔断的错误数：`ErrorAmountThreshold=20`。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      },
      "CircuitBreaking": {
        "MinRequestAmount": 10,
        "StatTimeWindow": "60",
        "ErrorAmountThreshold": "20",
        "DegradedTimeWindow": 10,
        "DegradedStatusCode": 503,
        "DegradedResponseContent": "Service is temporarily unavailable."
      }
    }
  }
}
```

还是保持服务端的 30% 错误率，再次测试可以看到 20 个 `500` 错误影响之后触发了熔断，返回了预设的 `503` 错误。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?status\=500:30

IP addresses distribution:
127.0.0.1:8080: 60
Code 200 : 40 (40.0 %)
Code 500 : 20 (20.0 %)
Code 503 : 40 (40.0 %)
```

### 错误响应占比触发熔断

接下来我们测试下错误响应占比的触发策略，将 `ErrorAmountThreshold=20`，修改为 `ErrorRatioThreshold=0.25`。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      },
      "CircuitBreaking": {
        "MinRequestAmount": 10,
        "StatTimeWindow": "60",
        "ErrorRatioThreshold": "0.25",
        "DegradedTimeWindow": 10,
        "DegradedStatusCode": 503,
        "DegradedResponseContent": "Service is temporarily unavailable."
      }
    }
  }
}
```

再次测试，可以看到 30 个请求中，有 6 个请求返回了 `500` 的响应，达到了 25% 的熔断阈值，触发了熔断。

```shell
fortio load -quiet -c 10 -n 100 -p 50,71,90,95,99 http://localhost:8080/echo\?status\=500:30

IP addresses distribution:
127.0.0.1:8080: 85
Code 200 : 24 (24.0 %)
Code 500 : 6 (6.0 %)
Code 503 : 70 (70.0 %)
```
