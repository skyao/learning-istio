---
date: 2018-09-29T20:00:00+08:00
title: Galley
weight: 600
description : "介绍istio Galley模块"
---

Istio Galley模块

### 介绍

Galley是Istio的顶级组件，负责配置摄取，处理和分发，是 Istio 1.0 中新引入的组件。



[Istio  官方文档](<https://istio.io/docs/concepts/what-is-istio/>) 中对 Galley 的概括介绍：

> Galley is Istio’s configuration validation, ingestion, processing and distribution component. It is responsible for insulating the rest of the Istio components from the details of obtaining user configuration from the underlying platform (e.g. Kubernetes).
>
> Galley 代表其他的 Istio 控制平面组件，用来验证用户编写的 Istio API 配置。随着时间的推移，Galley 将接管 Istio 获取配置、 处理和分配组件的顶级责任。它将负责将其他的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节中隔离开来。

Galley负责将其余的Istio组件与从底层平台获取用户配置的细节隔离开来:

- 它包含用于收集配置的Kubernetes CRD侦听器
- 用于分发配置的MCP协议服务器实现
- 以及用于Kubernetes API Server进行预摄取(pre-ingest)验证的验证Web挂钩。



### Galley容器

Gally模块包含一个单容器 `galley`: 提供 istio 中的配置管理服务, 验证Istio的CRD 资源的合法性.

Galley监听端口为:

- `--server-address`： galley提供服务的地址， gRPC协议, 默认是tcp://0.0.0.0:9901
- `--validation-port`：提供验证crd合法性服务的端口, HTTPS协议，默认443.
- `--monitoringPort` ：监控端口，HTTP协议, 默认为 15014

以上端口通过k8s service`istio-galley`对外提供服务。



## Galley参考资料

- [Galley参考文档（中文翻译版本）](https://istio.io/zh/docs/reference/commands/galley/)
- [配置验证 Webhook @ 官方文档](https://istio.io/zh/help/ops/setup/validation/)
- [Istio Helm Chart 详解 - Galley](https://blog.fleeto.us/post/istio-helm-deep-dive-galley/)