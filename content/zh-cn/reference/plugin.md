---
title: "插件"
description: "FGW 目前支持的插件列表，以及插件链。"
weight: 2
---

## 插件链

### HTTP 路由

HTTP 插件链，处理 HTTP 流量。

```json
{
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
  ]
}
```

### HTTPS 路由

HTTPS 插件链，处理 HTTPS 流量。与 HTTP 插件链相比，增加了插件 `common/tls-termination.js`。

```json
{
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
  ]
}
```

### TLS 透传

用于 [TLS 透传](/features/tls/passthrough/)场景的插件链。

```json
{
  "TLSPassthrough": [
    "common/access-control.js",
    "common/ratelimit.js",
    "tls/passthrough.js",
    "common/consumer.js"
  ]
}
```

### TLS 卸载

用于 [TLS 卸载](/features/tls/termination/)的插件链。

{
  "TLSTerminate": [
    "common/access-control.js",
    "common/ratelimit.js",
    "common/tls-termination.js",
    "common/consumer.js",
    "tls/forward.js"
  ]
}

### TCP 路由

```json
{
  "TCPRoute": [
    "common/access-control.js",
    "common/ratelimit.js",
    "tcp/forward.js"
  ]
}
```

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