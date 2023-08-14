---
title: "HTTP 头部控制"
description: "在 FGW 的配置中，HTTP 头部控制功能允许你微调传入和传出的请求和响应头部。本文档将介绍 FGW 的 HTTP 头部控制功能。"
weight: 8
---

## 介绍

在日常的网络交互中，HTTP 头部扮演了非常重要的角色。它们可以传递关于请求或响应的各种信息，如身份验证、缓存控制、内容类型等。FGW 提供了 HTTP 头部控制功能，允许用户对传入和传出的请求和响应头部进行精确的控制，以满足各种安全、性能和业务需求。

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 后端服务
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

这里的后端服务，可以使用 Pipy 来模拟，执行下面的命令即可。

```shell
pipy -e "pipy().listen(8081).serveHTTP(msg => new Message({ 'status': 200, headers: { 'content-type': 'application/json', 'server': 'Pipy', 'token': 'CONFIDENTIAL'} }, JSON.encode({ headers: msg.head })))"
```

模拟的服务会返回如下信息，在消息体中返回原始请求的所有头部信息。

```shell
curl -i http://localhost:8080/
HTTP/1.1 200 OK
content-type: application/json
server: Pipy
content-length: 200
connection: keep-alive

{"headers":{"protocol":"HTTP/1.1","headers":{"host":"localhost:8080","user-agent":"Apache-HttpClient/x.y.z","accept":"*/*","client-id":"client-1","connection":"keep-alive"},"method":"GET","path":"/"}}
```

## 配置说明

FGW 的 HTTP 头部控制功能分为 [请求头部控制 `RequestHeaderModifier`](/reference/configuration/#4131-requestheadermodifier) 和响应头部控制 [`ResponseHeaderModifier`](/reference/configuration/#4132-responseheadermodifier)，使用二者都可以对 HTTP 头部进行增加、删除、修改三种操作。

- `set`: 设置 HTTP header。此字段用于设定或修改 HTTP 头部的值。如果指定的头部已存在，则其值将被替换为新值；如果不存在，则会添加此头部。它是一个可选字段，其值是一个对象列表，其中每个对象都包含“name”和“value”两个键。
- `add`: 增加 HTTP header。此字段用于添加新的 HTTP 头部，而不会影响到原有的头部。如果指定的头部已存在，新的头部将被添加，而不会替换已有的头部。它是一个可选字段，其值格式与 `set` 字段相同。
- `remove`: 删除 HTTP header。此字段用于从 HTTP 请求或响应中删除指定的头部。这对于确保某些敏感或不必要的头部不被暴露给客户端或后端服务是很有用的。它是一个可选字段，其值是一个字符串列表，代表要删除的头部名称。

### 示例

```json
{
  "Filters": [
    {
      "Type": "RequestHeaderModifier",
      "set": [
        {
          "name": "host",
          "value": "set-bar"
        }
      ],
      "add": [
        {
          "name": "accept",
          "value": "xxx"
        }
      ],
      "remove": [
        "user-agent",
        "my-header4"
      ]
    },
    {
      "Type": "ResponseHeaderModifier",
      "set": [
        {
          "name": "dummy1",
          "value": "set-bar"
        }
      ],
      "add": [
        {
          "name": "dummy2",
          "value": "add,baz"
        }
      ],
      "remove": [
        "dummy3",
        "my-header8"
      ]
    }
  ]
}
```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，修改服务的断点列表为我们的后端服务地址 `127.0.0.1:8081`；然后根据 [HTTP 插件链配置](/reference/plugin/#http-路由)，将 `filter/header-modifier.js` 插件配置在相应的配置。

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
      "filter/header-modifier.js",
      "http/forward.js",
      "http/default.js"
    ]
  }
}
```

通过负载均衡访问后端服务。

```shell
curl -i http://localhost:8080
HTTP/1.1 200 OK
content-type: application/json
server: Pipy
token: CONFIDENTIAL
content-length: 164
connection: keep-alive

{"headers":{"protocol":"HTTP/1.1","headers":{"host":"localhost:8080","user-agent":"curl/8.1.2","accept":"*/*","connection":"keep-alive"},"method":"GET","path":"/"}}
```

### HTTP 头部控制配置

看到上面的结果后，我们希望对后端服务隐藏客户端的信息，比如将请求中的 `user-agent` 替换为 `Apache-HttpClient/x.y.z`；增加 `client-id` 信息 `client-1`；同时为了安全期间，避免服务端的 `token` 泄露，并模拟服务端的 `server` 信息为 `Tomcat Server`。

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
            "Type": "RequestHeaderModifier",
            "set": [
              {
                "name": "user-agent",
                "value": "Apache-HttpClient/x.y.z"
              }
            ],
            "add": [
              {
                "name": "client-id",
                "value": "client-1"
              }
            ]
          },
          {
            "Type": "ResponseHeaderModifier",
            "set": [
              {
                "name": "server",
                "value": "Tomcat Server"
              }
            ],
            "remove": [
              "token"
            ]
          }
        ]
      }
    }
  }
}
```

再次通过负载均衡访问后端服务，对应的头部信息已经做了修改。

```shell
curl -i http://localhost:8080/
HTTP/1.1 200 OK
content-type: application/json
server: Tomcat Server
content-length: 200
connection: keep-alive

{"headers":{"protocol":"HTTP/1.1","headers":{"host":"localhost:8080","user-agent":"Apache-HttpClient/x.y.z","accept":"*/*","client-id":"client-1","connection":"keep-alive"},"method":"GET","path":"/"}}
```
