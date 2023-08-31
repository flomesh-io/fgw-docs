---
title: "TLS 透传"
description: "TLS透传是指负载均衡器或网关不解密TLS流量，而是直接将加密数据传送到后端服务器，由后端服务器进行解密和处理。"
weight: 2
---

## 介绍

TLS 透传（TLS Passthrough）是代理服务器处理 TLS 请求的两种方式之一（另一种是 [TLS 卸载](/features/tls/termination/)）。在 TLS 透传模式下，代理不会解密来自客户端的 TLS 请求，而是将其传递到上游服务器进行解密，这就意味着数据在通过代理的时候保持加密的状态，以此来保证重要和敏感数据的安全性。

### TLS 透传的优点

* 由于数据不在代理上解密，而是以加密的方式转发到上游服务器，数据可以免受网络攻击。
* 加密数据未经解密到达上游服务，确保了数据的机密性。
* 这也是代理配置 TLS 的最简单方法。

### TLS 透传的缺点

  * 流量中可能会恶意代码，这些代码将直接到达后端服务器。
  * 在 TLS 透传过程中，无法切换服务器。
  * 无法做 7 层流量处理。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- TLS 后端服务

> TLS 的后端服务，本文档使用 `https://httpbin.org` 作为后端服务。

## 配置说明

### 示例

## 配置

1. 设置监听端口的协议，参考 [监听端口配置文档](/reference/configuration/#2-监听端口配置listeners)，我们选择 `TLS` 作为端口的协议。

```json
{
  "Listeners": [
    {
      "Protocol": "TLS",
      "Port": 8443,
      "TLS": {}
    }
  ]
}
```

2. 作为使用 TLS 协议的端口，需要对其 TLS 进行配置。根据 [TLS 配置文档](/reference/configuration/#22-tls)，功能模式 `TLSModeType` 支持 `Terminate` 和 `Passthrough` 两种类型。`Terminate` 在 [HTTPS 负载均衡](/features/tls/termination/) 有使用，这里我们使用 `Passthrough` 类型。

```json
{
  "TLS": {
    "TLSModeType": "Passthrough"
  }
}
```

3. TLS 透传是工作在 L4 上，配置路由规则时使用 [TLS 协议端口号 Passthrough 路由规则配置](/reference/configuration/#33-端口号配置protocol-tls-tlsmodetype-passthrough-的配置格式)，键值分别为上游的域名地址（或域名地址 + 端口）。这里，我们使用配置 `"httpbin.org": "httpbin.org"`。

```json
{
  "RouteRules": {
    "8443": {
      "httpbin.org": "httpbin.org"
    }
  }
}
```

4. 还有插件链的配置，我们选择用于 [TLS 透传的插件链](/reference/plugin/#tls-透传) `TLSPassthrough`。

```json
{
  "Chains": {
    "TLSPassthrough": [
      "tls/passthrough.js",
      "common/consumer.js"
    ]
  }
}
```

5. 最终得到完整的配置。

```json
{
  "Listeners": [
    {
      "Protocol": "TLS",
      "Port": 8443,
      "TLS": {
        "TLSModeType": "Passthrough"
      }
    }
  ],
  "RouteRules": {
    "8443": {
      "httpbin.org": "httpbin.org"
    }
  },
  "Chains": {
    "TLSPassthrough": [
      "tls/passthrough.js",
      "common/consumer.js"
    ]
  }
}

```

测试时，可以执行下面的命令向代理服务器发送请求。

```shell
curl https://httpbin.org/get --connect-to httpbin.org:443:127.0.0.1:8443
```
