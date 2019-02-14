---
date: 2018-09-29T20:00:00+08:00
title: DestinationRule
weight: 412
menu:
  main:
    parent: "crd-network"
description : "DestinationRule CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#destinationrule

`DestinationRule` 所定义的策略，决定了经过路由处理之后的流量的访问策略。这些策略中可以定义负载均衡配置、连接池尺寸以及外部检测（用于在负载均衡池中对不健康主机进行识别和驱逐）配置。

| 字段            | 类型                                                         | 描述                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `host`          | `string`                                                     | 必要字段。目标服务的名称。流量目标对应的服务，会在在平台的服务注册表（例如 Kubernetes 服务和 Consul 服务）以及 `ServiceEntry` 注册中进行查找，如果查找失败，则丢弃流量。**Kubernetes 用户注意：当使用服务的短名称时（例如使用 reviews，而不是 reviews.default.svc.cluster.local），Istio 会根据规则所在的命名空间来处理这一名称，而非服务所在的命名空间。假设 default 命名空间的一条规则中包含了一个 reivews 的 host 引用，就会被视为 reviews.default.svc.cluster.local，而不会考虑 reviews 服务所在的命名空间。为了避免可能的错误配置，建议使用 FQDN 来进行服务引用。** |
| `trafficPolicy` | `TrafficPolicy` | 流量策略（负载均衡策略、间接池尺寸和外部检测）。             |
| `subsets`       | `Subset` | 一个或多个服务版本。在子集的级别可以覆盖服务一级的流量策略定义。 |

