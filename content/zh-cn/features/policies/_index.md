---
title: "策略"
description: "本系列文档将介绍 FGW 高级策略的使用。"
weight: 10
---

Flomesh Gateway (FGW) 提供了一系列先进的负载均衡策略，以确保流量在后端服务之间有效、安全地分发。这些策略包括熔断、限流、超时、重试、重定向、路径重写、HTTP 头修改、会话保持和主动健康检查等。

本文档将深入解释这些策略的功能以及如何配置和使用。

这里介绍的部分策略与端点 `Endpoints` 一样，都是服务维度的配置。

- [熔断](/features/policies/circuit-breaking/)
- [会话保持](/features/policies/session-sticky/)
- [健康检查](/features/policies/healthcheck/)
- [重试](/features/policies/retry/)
- [HTTP 头部控制](/features/policies/header-manipulate/)
- [请求重定向](/features/policies/url-redirecting/)
- [路径重写](/features/policies/url-rewrite/)
- [URL 重写](/features/policies/rate-limiting/)
- [会话保持](/features/policies/session-sticky/)
- [流量镜像](/features/policies/request-mirror)

当然还有粒度更加灵活的策略，如 [限流]()，可以作用于域名和路由的粒度。

### 示例

```json
{
  "RouteRules": {
    "PORT": {
      "HOST": {
        "RouteType": "HTTP",
        "Matches": [
          {
            "Path": {},
            "BackendService": {},
            "RateLimit": {}
          }
        ],
        "RateLimit": {}
      }
    }
  },
  "Services": {
    "backendService": {
      "Endpoints": {},
      "CircuitBreaking": {},
      "Filters": [
        {
          "Type": "RequestHeaderModifier"
        },
        {
          "Type": "ResponseHeaderModifier"
        },
        {
          "Type": "RequestMirror"
        },
        {
          "Type": "RequestRedirect"
        },
        {
          "Type": "HTTPURLRewriteFilter"
        }
      ],
      "MaxRequestsPerConnection": 1,
      "MaxPendingRequests": 1,
      "RetryPolicy": {},
      "HealthCheck": {},
      "StickyCookieName": "COOKIE_NAME",
      "StickyCookieExpires": 3600
    }
  }
}
```
