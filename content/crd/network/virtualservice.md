---
date: 2018-09-29T20:00:00+08:00
title: VirtualService
weight: 2011
menu:
  main:
    parent: "crd-network"
description : "VirtualService CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice

`VirtualService` 定义了一系列针对指定服务的流量路由规则。每个路由规则都针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务（或者目标服务的子集或版本）。

匹配规则中还包含了对流量发起方的定义，这样一来，规则还可以针对特定客户上下文进行定制。


| 字段       | 类型                                                         | 描述                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `hosts`    | `string[]`                                                   | 必要字段：流量的目标主机。可以是带有通配符前缀的 DNS 名称，也可以是 IP 地址。根据所在平台情况，还可能使用短名称来代替 FQDN。这种场景下，短名称到 FQDN 的具体转换过程是要靠下层平台完成的。**一个主机名只能在一个 VirtualService 中定义。**同一个 `VirtualService` 中可以用于控制多个 HTTP 和 TCP 端口的流量属性。 Kubernetes 用户注意：当使用服务的短名称时（例如使用 `reviews`，而不是 `reviews.default.svc.cluster.local`），Istio 会根据规则所在的命名空间来处理这一名称，而非服务所在的命名空间。假设 “default” 命名空间的一条规则中包含了一个 `reviews` 的 `host`引用，就会被视为 `reviews.default.svc.cluster.local`，而不会考虑 `reviews` 服务所在的命名空间。**为了避免可能的错误配置，建议使用 FQDN 来进行服务引用。** `hosts` 字段对 HTTP 和 TCP 服务都是有效的。网格中的服务也就是在服务注册表中注册的服务，必须使用他们的注册名进行引用；只有 `Gateway` 定义的服务才可以使用 IP 地址。 |
| `gateways` | `string[]`                                                   | `Gateway` 名称列表，Sidecar 会据此使用路由。`VirtualService` 对象可以用于网格中的 Sidecar，也可以用于一个或多个 `Gateway`。这里公开的选择条件可以在协议相关的路由过滤条件中进行覆盖。保留字 `mesh` 用来指代网格中的所有 Sidecar。当这一字段被省略时，就会使用缺省值（`mesh`），也就是针对网格中的所有 Sidecar 生效。如果提供了 `gateways` 字段，这一规则就只会应用到声明的 `Gateway` 之中。要让规则同时对 `Gateway` 和网格内服务生效，需要显式的将 `mesh` 加入 `gateways` 列表。 |
| `http`     | `HTTPRoute[]` | HTTP 流量规则的有序列表。这个列表对名称前缀为 `http-`、`http2-`、`grpc-` 的服务端口，或者协议为 `HTTP`、`HTTP2`、`GRPC` 以及终结的 TLS，另外还有使用 `HTTP`、`HTTP2` 以及 `GRPC` 协议的 `ServiceEntry` 都是有效的。进入流量会使用匹配到的第一条规则。 |
| `tls`      | `TLSRoute[]` | 一个有序列表，对应的是透传 TLS 和 HTTPS 流量。路由过程通常利用 `ClientHello` 消息中的 SNI 来完成。TLS 路由通常应用在 `https-`、`tls-` 前缀的平台服务端口，或者经 `Gateway` 透传的 HTTPS、TLS 协议端口，以及使用 HTTPS 或者 TLS 协议的 `ServiceEntry` 端口上。**注意：没有关联 VirtualService 的 https- 或者 tls- 端口流量会被视为透传 TCP 流量。** |
| `tcp`      | `TCPRoute[] | 一个针对透传 TCP 流量的有序路由列表。TCP 路由对所有 HTTP 和 TLS 之外的端口生效。进入流量会使用匹配到的第一条规则。 |

