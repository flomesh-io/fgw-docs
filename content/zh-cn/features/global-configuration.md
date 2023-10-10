---
title: "全局配置"
description: "介绍 FGW 的全局配置，控制 FGW 的核心行为和处理流程"
weight: 1
draft: false
---

## 介绍

全局配置项是用来管理和调整 FGW 的核心行为和处理流程，这些配置能在 FGW 的各个功能和模块中被应用，支持如日志管理、网络超时设置、请求处理和健康检查等各种全局行为的定制化设置。

## 配置说明

下面是对[全局配置](/reference/configuration/#1-全局配置configs)的详细说明：

- `EnableDebug`：激活或禁用调试日志输出。当此选项设置为 `true` 时，系统将记录详细的调试信息，这对于开发和故障排查非常有价值。需要注意的是，持续的日志记录可能会对系统性能产生影响。

- `StripAnyHostPort`：控制是否剥离 HTTP 请求头中 Host 字段的端口信息。当此项为 `true` 时，例如，`Host: www.aaa.com:8080` 会被视作 `Host: www.aaa.com`，这有助于简化主机名的处理和路由。

- `DefaultPassthroughUpstreamPort`：在 [TLS 透传](/features/tls/passthrough/) 模式中，默认使用此端口号来转发流量，尤其是在上游服务器端口未被明确指定的情况下。它有助于确保在缺乏详细配置信息的情况下，流量能够被正确转发到目标。

- `SocketTimeout`：定义网络 socket 操作的超时时长，单位为 **秒**，默认值是 **60**。这是系统等待网络响应的最大时间阈值，通过这一设置可以防止网络问题导致的资源挂起。

- `PidFile`：指定一个文件路径，系统将在此文件中写入 Pipy 进程的 ID。这在进行进程管理和监视时可能会非常有用。

- `ResourceUsage`：控制是否收集及如何收集 Pipy 进程的资源使用数据，如 CPU 使用率和内存消耗。这些数据可用于监视系统的健康状态和进行性能优化。
  - `ScrapeInterval`: 定义了 CPU 和内存使用率的数据采集时间间隔，单位是秒。设置这个值可以帮助系统以一定的频率检查资源使用情况，从而更有效地监视资源利用率。
  - `StorageAddress`: 指定存储 CPU 和内存使用率数据的 REST URL。在此 URL 定义的位置，采集到的数据将被存储和进一步处理或分析。如果这个字段没有被配置，数据将不被存储。
  - `Authorization`: 包含访问上述 REST URL 所需的 Basic 认证信息。这在目标存储位置要求身份验证时是必要的，以确保数据的安全传输和存储。如果没有启用认证，此字段可以省略。

- `HealthCheckLog`：确定如何记录和存储系统的健康检查日志。该设置指定了存储这些日志的服务器或存储解决方案的详细信息。
  - `StorageAddress`: 用于指定一个 REST URL，用于存储健康检查日志信息。通过将日志信息发送到此 URL，可以方便的对健康检查的结果进行保存和进一步分析。若此配置项未设置，健康检查日志将不被外部存储。
  - `Authorization`: 提供访问上述 REST URL所需的 Basic 认证信息。此配置确保健康检查日志数据的安全传输和存储。在没有启用目标URL的认证机制的情况下，此配置项可省略。

- `Gzip`：包含控制如何压缩静态文本文件的参数。这些设置影响如何对静态内容应用 Gzip 压缩，有助于减少传输的数据量和提高响应速度。详细说明，请参考 [gzip 压缩](#gzip-压缩)

- `ProxyRedirect`：定义了如何重写上游服务器在 HTTP 响应（例如 Location 或 Refresh 头）中的 URL。这允许在必要时修改重定向 URL，以确保它们满足下游客户端的需求。详细说明，请参考[重定向响应控制](#重定向响应控制)

- `ErrorPage`：允许为特定的 HTTP 错误代码定义定制的错误页面或链接。当 FGW 检测到定义的错误代码时，它将返回与之关联的自定义页面或链接，以提供更丰富或友好的错误信息。详细说明，请参考[自定义错误页面](#自定义错误页面)

### 示例

```json
{
  "Configs": {
    "DefaultPassthroughUpstreamPort": 8443,
    "EnableDebug": true,
    "StripAnyHostPort": true,
    "SocketTimeout": 30,
    "PidFile": "/var/log/pipy.pid",
    "ResourceUsage": {
      "ScrapeInterval": 60
    }
  }
}
```

## 功能

### gzip 压缩

在 Web 开发中，利用 gzip 压缩可以减小传输的数据量，从而提高页面加载速度、减轻服务器的负担、并在一定程度上减少网络带宽的消耗。通过下面的字段，可以对 gzip 压缩进行精细化控制。

- `DisableGzip`: 用于启用或禁用静态文本文件的 Gzip 压缩功能。若设置为 true，则不进行 Gzip 压缩；默认值为 `false`，表示启用 Gzip 压缩。
- `GzipMinLength`: 定义文本文件压缩的最小字节数。仅当文本文件的大小达到或超过此值时，文件会被压缩；默认值为 `1024`。
- `GzipHttpVersion`: 设定启用文本文件压缩的最低 HTTP 版本。只有满足或超过此 HTTP 版本的请求才会收到 Gzip 压缩的响应。
- `GzipTypes`: 指定了一组 content-type，仅对这些类型的文本文件进行 Gzip 压缩，以确保仅对可安全压缩的文件类型执行压缩操作。
- `GzipDisable`: 提供一个针对 `user-agent` 的禁用压缩设置。通过识别请求头中的 `user-agent` 信息，当其符合此处设置的模式时，Gzip 压缩将不会被执行。

```json
{
  "Configs": {
    "Gzip": {
      "GzipMinLength": 1024,
      "GzipTypes": [
        "text/css",
        "text/xml",
        "text/html",
        "text/plain",
        "application/xhtml+xml",
        "application/javascript"
      ]
    }
  }
}
```

### 重定向响应控制

该功能用于修改代理服务器传回的 HTTP 响应中的 `Location` 和 `Refresh` 头字段中的文本。这主要是用在 FGW 作为反向代理时，以确保重定向 URL 正确指向通过 Nginx 代理访问的地址，而不是直接指向后端服务器的地址。

比如下面的配置中，会对响应中的 `Location` 和 `Refresh` 头字段中内容的路径进行替换。

```json
{
  "Configs": {
    "ProxyRedirect": {
      "http://www.flomesh.com/aa": "http://$http_host/ab",
      "http://www.flomesh.com/a0": "/a1"
    }
  }
}
```

### 自定义错误页面

这个功能用于定义当服务器返回指定的错误代码时应显示的页面。这是一种定制错误页面的方法，以便在出现问题时向用户显示更友好或更有用的信息。

- `Error`: 定义了一组要被自定义错误页面覆盖的 HTTP 错误码。例如，[404, 500] 表示对这两种 HTTP 错误提供自定义页面响应。
- `Page`: 指定用于展示的自定义错误页面的名称或者 URL 地址。如果提供了完整的 URL 地址，将直接重定向至该 URL；若提供文件名，则需要结合`Directory`参数来找到文件的实际路径。
- `Directory`: 指定自定义错误页面文件的存储目录。仅当`Page`参数是一个文件名而非完整 URL 时适用。此路径将与`Page`参数一起用来检索自定义错误页面的完整文件路径。如果`Page`是一个完整的 URL，此参数将被忽略。

```json
{
  "Configs": {
    "ErrorPage": [
      {
        "Error": [
          404
        ],
        "Page": "404.html",
        "Directory": "pages/"
      },
      {
        "Error": [
          502
        ],
        "Page": "502.html",
        "Directory": "pages/"
      }
    ]
  }
}
```