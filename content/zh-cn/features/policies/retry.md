---
title: "重试"
description: "重试功能为架构提供了额外的稳定性和韧性。本文档将介绍 FGW 的重试功能"
weight: 3
---

## 介绍

在复杂的网络环境中，服务间的通信可能会因为各种原因失败。为了提高系统的可用性和韧性，FGW 提供了重试功能，允许在满足特定条件时自动重新发送失败的请求。

FGW 的重试功能为架构提供了额外的稳定性和韧性。通过合理地配置重试策略，可以有效地提高服务的可用性，确保在面对网络波动和服务暂时性问题时，请求仍有机会得到正确的处理。

### 重试策略的考虑因素

在配置重试策略时，应考虑以下几点：

1. **服务的敏感性**：对于某些服务，如支付、订单创建等，应谨慎配置重试策略，避免重复请求导致的数据不一致。
2. **服务的容量**：过度的重试可能会导致目标服务的资源耗尽。确保目标服务有足够的容量来处理重试请求。
3. **重试间隔**：为了防止因频繁的重试导致的资源耗尽或服务雪崩，建议设置一个合理的重试间隔。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 这里后端服务建议使用工具 [Fortio](https://fortio.org)。Fortio 作为服务器运行时，可以在请求时指定延迟、响应状态，及其分布，非常方便测试熔断场景。同时 Forito 还可以作为负载生成器来使用。
>
> 启动服务器可以使用命令 `fortio server -http-port 8082`。

## 配置说明

在 FGW 中重试是 [服务 Service 维度的配置](/reference/configuration/#41-服务-配置格式)。

- `RetryOn`：重试条件字段，定义了触发重试机制的条件。此字段用于确定在哪些特定条件下应该执行重试。例如，可以设置为特定的 HTTP 状态码，如“5xx”，这意味着当后端服务返回任何 5xx 错误时，都将触发重试。必须字段。
- `PerTryTimeout`：每次尝试的超时时间，定义了单次重试请求等待响应的最长时间。如果在这个时间内没有得到后端服务的响应，该次重试会被认为是失败的，进而触发下一次的重试（如果还有剩余的重试次数）。此字段在网络不稳定或后端服务响应延迟的场景中特别有用，确保不会因为一个长时间未响应的请求而浪费整个重试机会。可选字段。如果未设置，每次重试可能会采用全局或默认的超时时间。可选字段。
- `NumRetries`：重试次数，表示在放弃前尝试重新发送请求的次数。此字段对于控制可能的重试风暴非常有用。设置适当的值可以确保不会对后端服务产生过多的负担，同时也能给请求一个重新成功的机会。可选字段，如果未设置，可能需要默认值或固定重试次数。
- `RetryBackoffBaseInterval`：重试间隔，定义了两次重试之间的等待时间。此字段帮助确保在连续的重试之间有适当的冷却时间，从而避免在短时间内对后端服务产生大量请求。它可以基于固定时间间隔，或者根据特定算法（如指数退避）动态调整。可选字段。如果未设置，重试可能会立即进行，或使用默认的间隔。

### 示例

```json
{
  "RetryPolicy": {
    "RetryOn": "5xx",
    "PerTryTimeout": 5,
    "NumRetries": 5,
    "RetryBackoffBaseInterval": 1
  }
}
```

## 配置

### 负载均衡器配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的端点列表为我们的后端服务地址 `127.0.0.1:8082`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，重试的功能在 `http/forward.js"` 插件中实现。

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
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

使用 Fortio 负载生成工具：1 个并发使用 qps 10 发送 100 个请求；同时设置服务端 30% 的情况下返回 503 错误，实际测试在 30% 上下 5% 范围内浮动。

```shell
fortio load -quiet -c 1 -n 100 -qps 10 http://localhost:8080/echo\?status\=503:30

IP addresses distribution:
127.0.0.1:8080: 33
Code 200 : 68 (68.0 %)
Code 503 : 32 (32.0 %)
All done 100 calls (plus 0 warmup) 3.524 ms avg, 10.0 qps
```

### 配置重试策略

设置服务 `backendService1` 的重试策略：针对 `503` 错误进行重试、重试 `5` 次。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8082": {
          "Weight": 100
        }
      },
      "RetryPolicy": {
        "RetryOn": "503",
        "PerTryTimeout": 5,
        "NumRetries": 5,
        "RetryBackoffBaseInterval": 1
      }
    }
  }
}
```

设置完重试策略后，使用同样的负载进行测试。此时可以看到所有的请求均成功，但总耗时增长非常明显，实际的 qps 只有 2.1。因此重试多次，对客户端来说单个请求的耗时增加了。**所以，需要在设置策略的时候充分考虑服务的容量，避免错误的设置导致重试风暴压垮后端服务。**

```shell
fortio load -quiet -c 1 -n 100 -qps 10 http://localhost:8080/echo\?status\=503:30

IP addresses distribution:
127.0.0.1:8080: 10
Code 200 : 100 (100.0 %)
All done 100 calls (plus 0 warmup) 482.113 ms avg, 2.1 qps
```

假如我们修改一下生成负载的设置，使服务端同样概率下返回 `500` 的错误，而重试策略只针对 `503` 错误。

```shell
fortio load -quiet -c 1 -n 100 -qps 10 http://localhost:8080/echo\?status\=500:30

IP addresses distribution:
127.0.0.1:8080: 28
Code 200 : 72 (72.0 %)
Code 500 : 28 (28.0 %)
All done 100 calls (plus 0 warmup) 3.397 ms avg, 10.0 qps
```
