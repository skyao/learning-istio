---
date: 2018-09-30T16:00:00+08:00
title: kiali概述
weight: 2021
menu:
  main:
    parent: "kiali"
description : "kiali概述，介绍kiali的基本情况和功能列表"
---

> 备注：内容翻译自官方文档 [OVERVIEW](https://www.kiali.io/documentation/overview/)

## Kiali是什么?

Kiali为以下问题提供答案：Istio服务网格中有哪些微服务，它们是如何连接的？

![Demo Website](https://www.kiali.io/images/documentation/overview/kiali.png)

微服务架构将单体结构分解成组合在一起的许多小块。已经出现了用于保护服务之间的通信的模式，例如容错（通过超时，重试，断路等），以及分布式跟踪以便能够看到调用的去向。

服务网格现在可以在平台级别上提供这些服务，并使应用程序编写者从这些任务中解放出来。路由决策在网格级别完成。

Kiali与Istio合作， 在 OpenShift 或者 Kubernetes 中可视化服务网格拓扑，熔断器或请求率等功能。它在不同层面提供了网格组件的见解，从抽象的应用程序到服务和工作负载。

Kiali还包括Jaeger Tracing，可以提供开箱即用的分布式跟踪功能。

## Kiali有什么作用？

### 图形视图

Kiali提供命名空间的实时交互式图形视图，能够显示多个级别上（应用程序，版本，工作负载）的交互，以及所选图形节点或边缘上的上下文信息和图表。

![Demo Website](https://www.kiali.io/images/documentation/overview/graph-view.png)

### 应用

Applications 菜单项显示在我们的环境中运行的所有应用程序。

![Demo Website](https://www.kiali.io/images/documentation/overview/app-list.png)

备注：在前几个服务前，有istio的帆船图标，表示这是istio项目。

Kiali提供与应用程序相关的详细信息，例如其健康状况或工作负载列表。健康摘要在提示中带有多个指标的详细信息。

![Demo Website](https://www.kiali.io/images/documentation/overview/app-view-info.png)

Kiali还显示与应用程序关联的Istio指标。![Demo Website](https://www.kiali.io/images/documentation/overview/app-metrics.png)

### 工作负载

Workloads 菜单项显示工作负载列表及其健康状况，错误率和标签验证。

![Demo Website](https://www.kiali.io/images/documentation/overview/workload-list.png)

通过选择工作负载，相关信息与相关的pod和服务一起显示。

![Demo Website](https://www.kiali.io/images/documentation/overview/workload-view-pods.png)

### 服务

Services 菜单项显示服务列表及其健康状况和错误率。

选择单个服务时，其详细信息页面包括service IP，端口，端点，工作负载，destination rule，virtual service和更多详细信息。

显示此服务的Inbound/outbound指标，并在链接的Grafana仪表板中提供更详细的视图。

![Demo Website](https://www.kiali.io/images/documentation/overview/service-view.png)

### Istio配置

Istio Config 菜单项显示在用户环境中存在的所有可用Istio配置对象的列表。

![Demo Website](https://www.kiali.io/images/documentation/overview/istio-list.png)

可以查看特定Istio对象的yaml配置。

![Demo Website](https://www.kiali.io/images/documentation/overview/istio-yaml.png)

Kiali将高亮显示配置错误。

![Demo Website](https://www.kiali.io/images/documentation/overview/istio-yaml-validation.png)

### 分布式追踪

单击“Distributed Tracing”菜单项将打开一个带有Jaeger UI的新标签，用于跟踪服务。

> 备注：这里会跳转到 Jaeger 的界面。理解为 kiali 只是和 Jaeger 做了集成。