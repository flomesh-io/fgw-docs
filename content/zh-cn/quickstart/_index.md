---
title: "快速体验"
description: "快速体验 FGW"
weight: 2
---

从 GitHub 克隆 FGW 仓库。

```shell
git clone https://github.com/flomesh-io/fgw.git
```

进入到工程的根目录，运行 `pipy` 并指定入口文件 `./pjs/main.js`。

```shell
cd fgw
pipy ./pjs/main.js
```

使用工程中默认的配置（`./pjs/config.json`）启动，将监听 3 个端口：

- `8080`：代理的端口，将请求负载均衡到另外两个静态服务
- `8081`：从 `/var/www/html` 目录启动的静态服务
- `8082`：从 `static/www2` 目录启动的静态服务

> 本地 `/var/www/html` 目录不存在，因此当访问 `8081` 的服务时，会报 `404` 的错误。

```shell
curl -I localhost:8081

HTTP/1.1 404 Not Found
content-length: 0
connection: keep-alive

curl -I localhost:8082

HTTP/1.1 200 OK
content-type: text/html
content-length: 72
connection: keep-alive
```

当我们访问代理的 `8080` 端口时，即使上游的两个静态服务权重都是 `50`，响应都是 `200`。

```shell
curl -I localhost:8080
HTTP/1.1 200 OK
content-type: text/html
content-length: 72
set-cookie: _srv_id=3021372512864600; path=/; expires=Fri,  4 Aug 2023 02:23:59 GMT; max-age=3600
connection: keep-alive
```

这是因为配置了[健康检查](/features/healthcheck/)，FGW 会主动对上游进行健康检查，响应状态码为 `200` 的上游才被认为是健康的，否则将其从负载均衡列表中移除。

```json
{
  "Services": {
    "backendService1": {
      "HealthCheck": {
        "Interval": 10,
        "MaxFails": 3,
        "FailTimeout": 30,
        "Uri": "/",
        "Matches": [
          {
            "Type": "status",
            "Value": "200"
          }
        ]
      }
    }
  }
}
```