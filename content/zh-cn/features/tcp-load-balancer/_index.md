---
title: "TCP 负载均衡"
description: "本文档将介绍如何配置 FGW 代理和负载均衡 TCP 流量。"
weight: 1
---

## 介绍

在 L4 负载均衡过程中，FGW 主要基于网络层和传输层信息，如 IP 地址和端口号，来决定将流量分配到哪个后端服务器。这种处理方式使得 FGW 可以快速地进行决策并将流量转发给适当的服务器，从而提高了整体的网络性能。

如果要负载均衡 HTTP 流量，请参考文档 [L7 负载均衡](/features/http-load-balancer)。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务

## 配置

1. 要使用 L4 负载均衡，首先我们需要将监听端口的协议设置为 `TCP`，参考文档[监听端口配置](/reference/configuration/#2-监听端口配置listeners)。

```json
{
  "Listeners": [
    {
      "Protocol": "TCP",
      "Port": 8080
    }
  ]
}
```

2. 接下来是设置端口 `8080` 的路由规则，参考文档 [TCP 协议端口号路由规则配置](/reference/configuration/#32-端口号配置protocol-为-tcp-的配置格式)。


```json
{
  "RouteRules": {
    "8080": {"backendService1":100}
  }
}
```

3. 配置服务端点，参考文档[服务配置](/reference/configuration/#4服务配置services)。

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
      }
    }
  }
}
```

4. 不要忘记插件链的配置，本文档主要介绍 TCP 流量的负载均衡，这里只需要引入 `TCPRoute` 插件链和 `tcp/forward.js` 插件即可。完整的插件配置可以参考文档[完整的插件配置](/reference/plugin/#完整配置)。

```json
"Chains": {
    "TCPRoute": [
      "tcp/forward.js"
    ]
  }
```

5. 最后完整的配置如下，使用其替换 FGW 工程中的 `pjs/config.json`。

```json
{
  "Listeners": [
    {
      "Protocol": "TCP",
      "Port": 8080
    }
  ],
  "RouteRules": {
    "8080": {
      "backendService1": {
        "Weight": 100
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
    "TCPRoute": [
      "tcp/forward.js"
    ]
  }
}
```

多次访问负载均衡器的 `8080` 端口，可以看到轮流返回两个后端服务的响应。