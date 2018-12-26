---
date: 2018-09-29T20:00:00+08:00
title: ServiceEntry
weight: 2014
menu:
  main:
    parent: "crd-network"
description : "ServiceEntry CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#serviceentry

`ServiceEntry` 能够在 Istio 内部的服务注册表中加入额外的条目，从而让网格中自动发现的服务能够访问和路由到这些手工加入的服务。`ServiceEntry` 描述了服务的属性（DNS 名称、VIP、端口、协议以及端点）。这类服务可能是网格外的 API，或者是处于网格内部但却不存在于平台的服务注册表中的条目（例如需要和 Kubernetes 服务沟通的一组虚拟机服务）。

| 字段         | 类型                                                         | 描述                                                         |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `hosts`      | `string[]`                                                   | 必要字段。绑定到 `ServiceEntry` 上的主机名。可以是一个带有通配符前缀的 DNS 名称。如果服务不是 HTTP 协议的，例如 `mongo`、TCP 以及 HTTPS 中，`hosts` 中的 DNS 名称会被忽略，这种情况下会使用 `endpoints` 中的 `address` 以及 `port` 来甄别调用目标。 |
| `addresss`   | `string[]`                                                   | 服务相关的虚拟 IP。可以是 CIDR 前缀。对 HTTP 服务来说，这一字段会被忽略，而会使用 HTTP 的 `HOST/Authority` Header。而对于非 HTTP 服务，例如 `mongo`、TCP 以及 HTTPS 中，这些主机会被忽略。如果指定了一个或者多个 IP 地址，对于在列表范围内的 IP 的访问会被判定为属于这一服务。如果地址字段为空，服务的鉴别就只能靠目标端口了。在这种情况下，被访问服务的端口一定不能和其他网格内的服务进行共享。换句话说，这里的 Sidecar 会简单的做为 TCP 代理，将特定端口的访问转发到指定目标端点的 IP、主机上去。就无法支持 Unix socket 了。 |
| `ports`      | `Port[]` | 必要字段。和外部服务关联的端口。如果 `endpoints` 是 Unix socket 地址，这里必须只有一个端口。 |
| `location`   | `ServiceEntry.Location` | 用于指定该服务的位置，属于网格内部还是外部。                 |
| `resolution` | `ServiceEntry.Resolution` | 必要字段。主机的服务发现模式。在没有附带 IP 地址的情况下，为 TCP 端口设置解析模式为 NONE 时必须小心。在这种情况下，对任何 IP 的指定端口的流量都是允许的（例如 `0.0.0.0:`）。 |
| `endpoints`  | `ServiceEntry.Endpoint[]` | 一个或者多个关联到这一服务的 `endpoint`。                    |