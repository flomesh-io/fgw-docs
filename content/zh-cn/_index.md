---
title: "Flomesh Gateway"
linkTitle: "Docs"
cascade:
  type: docs
---

Flomesh Gateway（简称 FGW）是一个开源的、基于 [Pipy](https://flomesh.io/pipy) 开发的全功能 API 网关和代理产品，旨在提供在微服务架构和其他网络应用中强大的流量管理功能。与其他知名的网关和代理解决方案（例如 Netflix Zuul、Spring Cloud Gateway、Nginx，以及 Kong）相比，FGW 提供了一系列的基础和高级功能，为现代的网络通信提供了丰富的支持。

FGW 的核心使用开源的可编程代理 [Pipy](https://flomesh.io/pipy)；提供丰富的可插拔的功能集，可以根据需求选择功能。

FGW 的设计还参考了 Kubernetes Gateway API，这意味着它可以很好地融入到 Kubernetes 生态系统中，并且可以方便地和其他云原生技术集成。它是 Flomesh Gateway API 实现的核心（即将发布），也将会成为 Flomesh 服务网格 FSM 的核心。