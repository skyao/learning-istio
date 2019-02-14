---
date: 2018-09-30T16:00:00+08:00
title: kiali词汇表
weight: 2022
menu:
  main:
    parent: "kiali"
description : "kiali词汇表"
---

> 备注：内容翻译自官方文档 [GLOSSARY](https://www.kiali.io/documentation/glossary/)

## 概念

### Application

Application是工作负载的逻辑分组，由用户应用于对象的application标签定义。 在Istio中，它由Label App定义。 [Istio标签要求](https://istio.io/docs/setup/kubernetes/spec-requirements/)。

#### Application Name

Application Name是在环境中部署的应用程序的名称。此名称由Workload上的`Label App`提供。

### Deployment

Deployment 是副本控制器，基于称为 Deployment 配置的用户定义模板。Deployment 可以是手动创建，也可以是响应触发的事件。

#### Istio object/configuration 类型

这是 Istio Config 中指定的类型。可以是以下任何类型：Gateway，VirtualService，DestinationRule，ServiceEntry，Rule，QuotaSpec 或 QuotaSpecBinding。

### Istio Sidecar

更多信息，请参阅[Istio Sidecar文档](https://istio.io/docs/reference/commands/sidecar-injector/)中的 [Istio Sidecar](https://www.kiali.io/documentation/glossary/concepts/#_istio_sidecar)  定义。

### Label

Label是用户创建的标记，用于标识一组对象。

empty [标签选择器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)（即，没有需求的选择器）选择集合中的每个对象。

null 标签选择器（仅可用于可选选择器字段）不选择任何对象。

例如，Istio使用Workload上的 Label App 和 Label Version 来指定版本和应用程序。

#### Label App

这是对象上的“app”标签。 有关更多信息，请参阅链接： [Istio标签要求](https://istio.io/docs/setup/kubernetes/spec-requirements/)。

#### Label Version

这是对象上的“version”标签。 有关更多信息，请参阅链接： [Istio标签要求](https://istio.io/docs/setup/kubernetes/spec-requirements/)。

### Namespace

Namespace旨在用于多个用户分布在多个团队或项目中的环境中。

Namespace是一种在多个用户之间划分群集资源的方法。

### Quota

有限或固定数量的资源。

### QuotaSpecBinding

Quota spec binding(配额规范绑定)用于将 mixer 配额资源名称绑定到代理。

### ReplicaSet

确保同时运行指定数量的pod副本。

### Service

Service是一种抽象，它定义了一组逻辑Pod，以及一个访问它们的策略。服务由标签确定。

#### Service Entry

有关更多信息，请参阅 [Istio Service Entry Documentation](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceEntry) 中的 [Service Entry](https://www.kiali.io/documentation/glossary/concepts/#_service_entry) 定义。

ServiceEntry在Istio中的定义补充如下：

`ServiceEntry` 能够在 Istio 内部的服务注册表中加入额外的条目，从而让网格中自动发现的服务能够访问和路由到这些手工加入的服务。`ServiceEntry` 描述了服务的属性（DNS 名称、VIP、端口、协议以及端点）。这类服务可能是网格外的 API，或者是处于网格内部但却不存在于平台的服务注册表中的条目（例如需要和 Kubernetes 服务沟通的一组虚拟机服务）。

详细内容见中文翻译版本 [`ServiceEntry`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#serviceentry) 。

### VirtualService

有关更多信息，请参阅 [Istio VirtualService Documentation](https://istio.io/docs/reference/config/istio.networking.v1alpha3/#VirtualService) 中的 [VirtualService](https://www.kiali.io/documentation/glossary/concepts/#_virtualservice) 定义。

VirtualService在Istio中的定义补充如下：

`VirtualService` 定义了一系列针对指定服务的流量路由规则。每个路由规则都针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务（或者目标服务的子集或版本）。

详细内容见中文翻译版本 [`VirtualService`](https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#virtualservice) 。

### Workload

或更多信息，请参阅 [Istio Glossary](https://istio.io/help/glossary/#workload) 中的工作负载定义。

工作负载在Istio中的定义补充如下：

运维部署的二进制文件，用于在Istio中提供某些功能。工作负载具有名称，名称空间和唯一ID。使用以下属性(attributes)则在策略和遥测配置中可以使用这些属性(properties)：

- `source.workload.name`, `source.workload.namespace`, `source.workload.uid`
- `destination.workload.name`, `destination.workload.namespace`, `destination.workload.uid`

在Kubernetes中，工作负载通常对应于Kubernetes deployment，而工作负载实例对应于deployment管理的单个Pod。

#### Workload Type

Workload的类型，Kiali目前支持 Deployment 类型。

## 网络



## 可观察性



