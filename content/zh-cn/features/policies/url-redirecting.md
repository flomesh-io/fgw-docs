---
title: "重定向"
description: "请求重定向是一种使客户端将其请求发送到另一个位置的方法。本文档将介绍 FGW 的请求重定向功能。"
weight: 9
---

## 介绍

请求重定向是一种常见的网络应用功能，它使得服务器能够告诉客户端：“你请求的资源已经移动到了另一个位置，请到新的位置去获取”。

随着互联网应用的复杂性增加，请求重定向变得越来越常见。FGW 的请求重定向功能提供了一个灵活而高效的方式来控制和管理这些重定向，确保流量能够正确并高效地流向正确的目的地。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 两个后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

> 用于健康检查的两个服务，可以使用 Pipy 快速模拟。

```shell
#origin
pipy -e "pipy().listen(8081).serveHTTP(new Message({status: 200},'Hello, world'))"
#target
pipy -e "pipy().listen(8082).serveHTTP(new Message({status: 200},'Hello, there'))"
```

```shell
curl http://localhost:8081
Hello, world
curl http://localhost:8082
Hello, there
```

## 配置说明

下面根据 [请求重定向的参考文档](/reference/configuration/#4133-requestredirect)，对各个字段就行说明。

- `scheme`：协议描述，此字段定义了重定向目标的协议类型。参考值：`http`、`https`。例如，若选择了 `https`，那么请求会被重定向到一个 HTTPS 协议的地址。
- `hostname`：重定向到的域名，此字段指定了重定向目标的域名或主机名。例如，设置为 `example.com`，则请求会被重定向到 `example.com`。
- `path`：重定向到的路径，此字段是必须的，它定义了重定向目标的特定路径。例如，若设定路径为 `/new-path`，请求将被重定向到指定的 `/new-path`。
- `statusCode`：重定向返回的状态码，此字段是必须的，它定义了服务器返回的 HTTP 状态码，通常用来告知客户端重定向的类型。

### 示例

```json
{
  "Filters": [
    {
      "Type": "RequestRedirect0",
      "scheme": "https",
      "hostname": "",
      "path": "/abc",
      "port": 8443,
      "statusCode": 301
    }
  ]
}
```

## 配置

### 负载均衡器配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081` ，重定向功能实现在 `filter/request-redirect.js` 中，需要将其添加到插件链中。

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
      "http/service.js",
      "filter/request-redirect.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

在服务的端点列表中，我们配置了 `127.0.0.1:8081`，通过负载均衡访问会返回 `Hello, world`。

```shell
curl http://localhost:8080
Hello, world
```

### 配置请求重定向

设置将请求重定向到 `8082` 的后端服务，将 `statusCode` 设置为 `301`。

```json
{
  "Services": {
    "backendService1": {
      "Endpoints": {
        "127.0.0.1:8081": {
          "Weight": 100
        },
        "Filters": [
          {
            "Type": "RequestRedirect",
            "scheme": "http",
            "path": "/",
            "port": 8082,
            "statusCode": 301
          }
        ]
      }
    }
  }
}
```

配置生效后，再次请求。

```shell
curl -i http://localhost:8080
HTTP/1.1 301 Moved Permanently
Location: http://localhost:8082/
content-length: 0
connection: keep-alive
```

借助 `-L` 参数，实现自动重定向。

```shell
curl -L http://localhost:8080
Hello, there
```
