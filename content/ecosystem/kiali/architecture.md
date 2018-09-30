---
date: 2018-09-30T17:00:00+08:00
title: kiali架构
weight: 723
menu:
  main:
    parent: "kiali"
keywords:
- kiali架构
description : "描述kiali的架构"
---

> 备注：内容翻译自官方文档 [ARCHITECTURE](https://www.kiali.io/documentation/architecture/)

Kiali由两个组件组成：在容器应用程序平台中运行的后端应用程序，以及面向用户的前端应用程序。此外，Kiali依赖于容器应用平台和Istio提供的外部服务和组件。

下图说明了Kiali及其交互中涉及的组件：

![Kiali architecture](https://www.kiali.io/images/documentation/architecture/architecture.png)

### Kiali后端

Kiali需要Istio。Istio是提供和控制服务网格的组件。虽然Kiali和Istio可以单独安装，但是Kiali依赖于Istio，如果Istio不存在则无法工作。

Kiali需要检索Istio数据和配置，这些是通过Prometheus和集群API公开的。这就是图表显示虚线的原因：表示间接依赖。

### Prometheus

Prometheus是Istio的依赖。启用Istio遥测后，度量数据将存储在Prometheus中。Kiali使用存储在Prometheus中的数据来计算网格拓扑，显示指标，计算健康状况，显示可能的问题等。

Kiali直接与Prometheus通信，并采用Istio Telemetery使用的数据模式。这对Kiali来说是一种硬依赖，如果没有，Kiali的大多数功能都无法运行。

目前，Kiali依赖于Istio的 [Istio’s default metrics](https://istio.io/docs/reference/config/policy-and-telemetry/metrics/)。确保始终使用这些默认指标。Kiali不支持自定义指标。如果您需要自定义指标或默认指标的变体，请使用自定义名称创建新的Istio指标。

### Cluster API

Kiali使用容器应用程序平台的API（cluster API）来获取和解析服务网格配置。

已知Kiali工作的容器应用平台是 [OKD](http://www.okd.io/) 和 [Kubernetes](http://kubernetes.io/)。Kiali也在研究这些平台的衍生产品。如果您想学习cluster API，请查看 [OKD REST API reference](https://docs.okd.io/latest/rest_api/index.html) 和 [Kubernetes API reference](https://kubernetes.io/docs/reference/kubernetes-api/)。

Kiali查询cluster API以检索诸如：namespace，service，deployment，pod和其他实体的定义。Kiali还会进行查询以解析不同群集实体之间的关系。

群集API还用于查询以检索Istio配置，例如：virtual services，destination rules，route rules，gateways，quotas等。

### Jaeger

Jaeger是可选的。如果可用，Kiali将能够将用户引导至Jaeger的跟踪数据。如果您需要此功能，请确保 [为Jaeger集成正确配置Kiali](https://github.com/kiali/kiali#jaeger)。

只有启用了 [Istio的分布式跟踪](https://istio.io/docs/tasks/telemetry/distributed-tracing/) ，才能使用跟踪数据。

### Grafana

Grafana是可选的。如果可用，Kiali的指标页面将显示一个链接，指导用户使用Grafana中的相同指标。如果您需要此功能，请确保 [为Grafana集成正确配置Kiali](https://github.com/kiali/kiali#grafana)。

Kiali具有基本的度量能力。它可以显示工作负载，应用和服务的默认Istio指标。它允许将一些分组应用于提供的指标并获取不同时间范围的指标。但是，Kiali不允许自定义视图，也不允许自定义Prometheus查询。如果您需要这些功能，则需要安装Grafana。如果需要，请按照Istio文档安装Grafana。

