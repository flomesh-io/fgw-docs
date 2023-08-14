---
title: "插件"
description: "FGW 目前支持的插件列表，以及插件链。"
weight: 2
---

## 插件链

### HTTP 路由

### HTTPS 路由

### TLS 透传

### TLS 卸载

### TCP 路由

## 完整配置

```json
{
  "Chains": {
    "HTTPRoute": [
      "common/access-control.js",
      "common/ratelimit.js",
      "common/consumer.js",
      "http/codec.js",
      "http/auth.js",
      "http/route.js",
      "http/service.js",
      "http/metrics.js",
      "http/tracing.js",
      "http/logging.js",
      "http/circuit-breaker.js",
      "http/throttle-domain.js",
      "http/throttle-route.js",
      "filter/request-redirect.js",
      "filter/header-modifier.js",
      "filter/url-rewrite.js",
      "http/forward.js",
      "http/default.js"
    ],
    "HTTPSRoute": [
      "common/access-control.js",
      "common/ratelimit.js",
      "common/tls-termination.js",
      "common/consumer.js",
      "http/codec.js",
      "http/auth.js",
      "http/route.js",
      "http/service.js",
      "http/metrics.js",
      "http/tracing.js",
      "http/logging.js",
      "http/circuit-breaker.js",
      "http/throttle-domain.js",
      "http/throttle-route.js",
      "filter/request-redirect.js",
      "filter/header-modifier.js",
      "filter/url-rewrite.js",
      "http/forward.js",
      "http/default.js"
    ],
    "TLSPassthrough": [
      "common/access-control.js",
      "common/ratelimit.js",
      "tls/passthrough.js",
      "common/consumer.js"
    ],
    "TLSTerminate": [
      "common/access-control.js",
      "common/ratelimit.js",
      "common/tls-termination.js",
      "common/consumer.js",
      "tls/forward.js"
    ],
    "TCPRoute": [
      "common/access-control.js",
      "common/ratelimit.js",
      "tcp/forward.js"
    ]
  }
}
```