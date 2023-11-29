---
title: "UDP 负载均衡"
description: "本文档将介绍如何配置 FGW 代理和负载均衡 TCP 流量。"
weight: 5
---

## 介绍

FGW 的 UDP 负载均衡功能针对基于 UDP 协议的网络流量提供了负载均衡解决方案。UDP 作为一个轻量级的、无连接的协议，通常用于视频流、在线游戏、VoIP 等应用，其中快速传输比完整性或顺序更重要。FGW 通过其 UDP 负载均衡功能，能够确保这些应用的流量在多个服务器之间得到有效分配，从而提高性能和可靠性。

UDP 负载均衡的配置与 [TCP](/features/tcp-load-balancer/) 的类似。

FGW 的 UDP 负载均衡配置涉及指定要分配流量的 UDP 服务端口，以及确定如何将流量分配到后端服务器。

通过精确配置这些参数，可以确保 UDP 负载均衡既高效又可靠，满足高性能应用的需求。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务

在这篇文档里我们将使用 [Pipy](https://flomesh.io/pipy/) 来运行简单的 UDP 应用。

```shell
pipy -e "pipy().listen(8078, {protocol: 'udp'}).replaceData(data => (console.log(data.toString()), new Data('You are requesting port ' + __inbound.localPort + ' \n')))" &
pipy -e "pipy().listen(8079, {protocol: 'udp'}).replaceData(data => (console.log(data.toString()), new Data('You are requesting port ' + __inbound.localPort + ' \n')))" &
```

## 配置

1. 要负载均衡 UDP 的请求，需要配置一个处理 UDP 协议的监听器，参考文档[监听端口配置](/reference/configuration/#2-监听端口配置listeners)。

```json
{
  "Listeners": [
    {
      "Protocol": "UDP",
      "Port": 8000
    }
  ]
}
```

2. 接下来要设置端口 `8000` 的路由规则，参考文档 [UDP 协议端口号路由规则配置](/reference/configuration/#32-端口号配置protocol-为-tcpudp-的配置格式)。

```json
{
  "RouteRules": {
    "8000": {"backendService1":100}
  }
}
```

3. 配置后端服务，参考文档[服务配置](/reference/configuration/#4-服务配置services)。后端服务的端点，配置为上面启动的两个 UDP 服务。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8078": {
          "Weight": 100
        },
        "127.0.0.1:8079": {
          "Weight": 100
        }
      }
    }
  }
}
```

5. 针对 UDP 请求的负载均衡，使用的插件同样简单。仅需引入 `UDPRoute` 插件链和 `udp/forward.js` 插件。完整的插件配置可以参考文档[完整的插件配置](/reference/plugin/#完整配置)。

```json
{
  "Chains": {
    "UDPRoute": [
      "udp/forward.js"
    ]
  }
}
```

6. 最后，可以得到完整的配置，替换 FGW 工程中的 `pjs/config.json`。

```json
{
  "Listeners": [
    {
      "Protocol": "UDP",
      "Port": 8000
    }
  ],
  "RouteRules": {
    "8000": {
      "backendService1": 100
    }
  },
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8078": {
          "Weight": 100
        },
        "127.0.0.1:8079": {
          "Weight": 100
        }
      }
    }
  },
  "Chains": {
    "UDPRoute": [
      "udp/forward.js"
    ]
  }
}
```

7. 测试

```shell
echo '' | nc -4u -w1 localhost 8000
You are requesting port 8079

echo '' | nc -4u -w1 localhost 8000
You are requesting port 8078

echo '' | nc -4u -w1 localhost 8000
You are requesting port 8079
```