---
date: 2018-09-29T20:00:00+08:00
title: Gateway
weight: 2013
menu:
  main:
    parent: "crd-network"
description : "Gateway CRD"
---

## 介绍

官方文档：

- https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Gateway
- https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway

`Gateway` 描述了一个负载均衡器，用于承载网格边缘的进入和发出连接。这一规范中描述了一系列开放端口，以及这些端口所使用的协议、负载均衡的 SNI 配置等内容。

### Gateway

| 字段       | 类型                                                         | 描述                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `servers`  | `Server`| 必要字段。`Server` 定义列表。                                |
| `selector` | `map<string, string>`                                        | 必要字段。用一个或多个标签来选择一组 Pod 或虚拟机，用于应用 `Gateway` 配置。标签选择的范围是平台相关的。例如在 Kubernetes 上，选择范围包含所有可达的命名空间。 |

### Server

| 字段    | 类型              | 描述                                                         |
| ------- | ----------------- | ------------------------------------------------------------ |
| `port`  | Port              | 必要字段。代理服务器监听的端口，用于接收连接。               |
| `hosts` | string[]          | 必要字段。`Gateway` 公开的主机名列表。最少要有一条记录。在通常的 HTTP 服务之外，也可以用于带有 SNI 的 TLS 服务。可以使用包含通配符前缀的域名，例如 `*.foo.com` 匹配 `bar.foo.com`，`*.com`匹配 `bar.foo.com` 以及 `example.com`。**注意**：绑定在 `Gateway` 上的 `VirtualService` 必须有一个或多个能够和 `Server` 中的 `hosts` 字段相匹配的主机名。匹配可以是完全匹配或是后缀匹配。例如 `server` 的 `hosts` 字段为 `*.example.com`，如果 `VirtualService` 的 `hosts` 字段定义为 `dev.example.com` 和 `prod.example.com`，就是可以匹配的；而如果`VirtualService` 的 `hosts` 字段是 `example.com` 或者 `newexample.com` 则无法匹配。 |
| `tls`   | Server.TLSOptions | 一组 TLS 相关的选项。这些选项可以把 http 请求重定向为 https，并且设置 TLS 的模式。 |