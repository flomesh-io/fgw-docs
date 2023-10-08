---
title: "负载均衡算法"
description: "本文档将介绍如何为服务指定负载均衡算法"
weight: 8
---

## 介绍

在微服务和 API 网关架构中，负载均衡是至关重要的，它确保每个服务实例都能平均地处理请求，同时也为高可用性和故障恢复提供了机制。FGW 提供了多种负载均衡算法，让可以根据业务需求和流量模式选择最适合的方法。

FGW 支持多种负载均衡算法，方便高效地分配流量，最大化资源利用率，提高服务的响应时间：

- **RoundRobinLoadBalancer**：这是最常见的负载均衡算法，请求将按顺序分配给每个服务实例。如果不特别指定，FGW 默认使用此算法。
- **HashingLoadBalancer**：根据请求的某些属性（如来源 IP 或请求头）计算哈希值，然后根据该哈希值将请求路由到特定的服务实例。这确保了相同的请求者或相同类型的请求总是被路由到同一服务实例。
- **LeastConnectionLoadBalancer**：这种算法会考虑每个服务实例的当前工作负载（连接数），并将新的请求分配给当前负载最小的实例，从而确保更均匀的资源利用。

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
pipy -e "pipy().listen(8082).serveHTTP(new Message({status: 200},'Hi, world'))"
```

## 配置说明

负载均衡算法（`Algorithm`）是服务级别（`Service`）的配置，可以参考 [服务配置文档](/reference/configuration/#41-服务-配置格式)。

配置方法很简单，只需在服务的 `Algorithm` 字段指定所需的算法即可。

### 示例

```json
{
  "Services": {
    "backendService1": {
      "Algorithm": "LeastWorkLoadBalancer",
      "Endpoints": {}
    }
  }
}
```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081` 和 `127.0.0.1:8082`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，重试的功能在 `http/forward.js"` 插件中实现。

为了便于测试，我们将两个端点的权重分别设为 `100` 和 `10`

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
          "Weight": 10
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

当不指定任何负载均衡算法时，默认使用 `RoundRobinLoadBalancer`。使用 `fortio load -quiet -c 55 -t 60s -qps 20 http://localhost:8080` 生成负载，并查看到两个后端的连接数。

可以看到在加权轮训算法下，代理会按照权重选择后端的示例建立连接。`50:5` 正好是两个后端实例的权重比 `100:10`。

```shell
netstat -lant | grep EST | awk '{print $5}' | grep 127.0.0.1 | grep -e "8081\|8082" | sort | uniq -c
  50 127.0.0.1.8081
   5 127.0.0.1.8082
```

### 配置哈希一致性均衡算法

在服务配置上通过 `Algorithm` 指定使用 `HashingLoadBalancer` 哈希一致性负载均衡算法。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 10
        }
      },
      "Algorithm": "HashingLoadBalancer"
    }
  }
}
```

待配置生效后，多次请求都可以获得相同的结果。

```shell
curl http://localhost:8080
Hello, world
curl http://localhost:8080
Hello, world
curl http://localhost:8080
Hello, world
```

同样在使用 `fortio load -quiet -c 55 -t 60s -qps 20 http://localhost:8080` 使用 55 个连接发送请求时，所有连接都连到了同一个后端实例。

```shell
netstat -lant | grep EST | awk '{print $5}' | grep 127.0.0.1 | grep -e "8081\|8082" | sort | uniq -c
  55 127.0.0.1.8082
```

### 配置最小连接数均衡算法

将均衡算法改为 `LeastWorkLoadBalancer`。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "127.0.0.1:8082": {
          "Weight": 10
        }
      },
      "Algorithm": "LeastConnectionLoadBalancer"
    }
  }
}
```

同样在使用 `fortio load -quiet -c 55 -t 60s -qps 20 http://localhost:8080` 使用 55 个连接发送请求时，到后端实例的连接数接近均衡。

```shell
netstat -lant | grep EST | awk '{print $5}' | grep 127.0.0.1 | grep -e "8081\|8082" | sort | uniq -c
  28 127.0.0.1.8081
  27 127.0.0.1.8082
```
