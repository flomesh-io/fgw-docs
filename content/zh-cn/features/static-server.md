---
title: "静态 Web 服务"
description: ""
weight: 13
---

## 介绍

FGW 的静态 Web 服务功能主要指的是 FGW 能够直接为客户端提供静态资源，如 HTML、CSS、JavaScript 文件、图像等，而无需转发请求到后端应用服务器。这可以减少不必要的网络延迟，提高静态资源的加载速度，同时也减轻了后端服务器的负担。

可以在 FGW 的配置中指定一个本地目录作为静态资源的存储位置。当 FGW 接收到请求时，可以根据请求的 URL 在资源目录中搜索匹配的文件，可以作为一个静态服务器运行。

FGW 能够识别多种静态文件类型，并根据文件类型发送适当的 `Content-Type` 响应头。这确保了客户端（如浏览器）能够正确地解析和显示资源。

## 前置条件

- Pipy（版本 >= {{< param min_pipy_version >}}）
- FGW Repo（版本 >= {{< param fgw_version >}}）
- 了解 [HTTP 负载均衡的配置](/features/http-load-balancer/)

## 配置说明

静态 Web 服务功能的配置非常简单，只需在 [配置端口的路由规则](/features/http-load-balancer/#配置端口的路由规则) 时，不指定后端服务，而是通过 `ServerRoot` 字段指定本地存放静态资源的目录即可。即从路由到后端服务，变为路由到本地资源。

- `ServerRoot` 字段指定了本地资源目录，可以使用绝对路径或相对路径。使用相对路径时，本地资源应该存放到当前 [代码库](/overview/concept/#代码库) 的目录中。
- `Index` 指定了默认文件的列表，当 URI 指向的是个目录时（以 `/` 结尾），会在对应的目录中查找默认文件。
- `TryFiles` 在许多 Web 服务器配置中都有类似的功能，它的主要目的是为了定义一个资源查找的优先级和顺序。在给定的静态 Web 服务配置中，它表示在请求某个 URI 时，服务器应该尝试按照 `TryFiles` 指定的顺序来查找和提供文件。如下面示例中会优先按照 URI 来查找文件，然后查找 URI 对应的子目录 `defaut` 的默认文件（`index.html` 或 `index.htm`），最后仍然找不到会返回 404。

### 示例

```json
{
  "RouteRules": {
    "8080": {
      "*": {
        "Matches": [
          {
            "ServerRoot": "/var/www/html",
            "Index": [
              "index.html",
              "index.htm"
            ],
            "TryFiles": [
              "$uri",
              "$uri/default/",
              "=404"
            ]
          }
        ]
      }
    }
  }
}

```

## 配置

### 负载均衡配置

我们借助在文档 [HTTP 负载均衡](/features/http-load-balancer/) 中的负载均衡器配置，去掉路径的路由匹配规则以及服务配置，改为静态服务配置。

在静态服务配置中，将本地资源目录设置为 `/tmp`，并设置默认文件列表。

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
            "ServerRoot": "/tmp",
            "Index": [
              "index.html",
              "index.htm"
            ]
          }
        ]
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

然后在 `/tmp` 目录中添加 `index.html` 文件。

```shell
cat > /tmp/index.html <<EOF
<!DOCTYPE html>  
<html>  
<body>  
<h1>hello</h1>  
<p>www1</p>  
</body>  
</html>
EOF
```

尝试访问 web 服务，可以返回配置的默认页面。

```shell
curl http://localhost:8080/
<!DOCTYPE html>
<html>
<body>
<h1>hello</h1>
<p>www1</p>
</body>
</html>
```
