---
date: 2018-09-29T20:00:00+08:00
title: EnvoyFilter
weight: 415
menu:
  main:
    parent: "crd-network"
description : "EnvoyFilter CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#envoyfilter

EnvoyFilter 对象描述了针对代理服务的过滤器，这些过滤器可以定制由 Istio Pilot 生成的代理配置。这一功能一定要谨慎使用。错误的配置内容一旦完成传播，可能会令整个服务网格进入瘫痪状态。

注 1：这一配置十分脆弱，因此不会有任何的向后兼容能力。这一配置是用于对 Istio 网络系统的内部实现进行变更的。

注 2：如果有多个 EnvoyFilter 绑定在同一个工作负载上，所有的配置会按照创建时间的顺序进行处理。如果多个配置之间存在冲突，会产生不可预知的后果。

| 字段             | 类型                                                         | 描述                                                         |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `workloadLabels` | `map<string, string>`                                        | 一个或多个标签，用于标识一组 Pod/虚拟机。这一组工作负载实例中的代理会被配置使用附加的过滤器配置。标签的搜索范围是平台相关的。例如在 Kubernetes 中，生效范围会包括所有可达的命名空间。如果省略这一字段，配置将会应用到网格中的所有 Envoy 代理实例中。注意：一个工作负载只应该使用一个 `EnvoyFilter`。如果多个 `EnvoyFilter` 被绑定到同一个工作负载上，会产生不可预测的行为。 |
| `filters`        | `EnvoyFilter.Filter[]` | 必要字段。要加入指定监听器之中的 Envoy 网络过滤器/HTTP 过滤器配置信息。当给 http 连接加入网络过滤器的时候，应该注意确保该过滤器应早于 `envoy.httpconnectionmanager`。 |