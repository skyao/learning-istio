---
date: 2018-09-29T20:00:00+08:00
title: ServiceDependency
weight: 416
menu:
  main:
    parent: "crd-network"
description : "ServiceDependency CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/docs/reference/config/istio.networking.v1alpha3/#ServiceDependency

`ServiceDependency`描述工作负载依赖的服务集合。换句话说，它描述了从给定工作负载的外出流量的属性。默认情况下，Istio建立的服务网格将具有完整的网格连接 - 即每个工作负载都拥有访问网格中的每个其他工作负载的代理配置。但是，大多数连接图在实践中都很稀疏。ServiceDependency提供了一种的方法来声明与每个工作负载关联的服务依赖关系，以便发送到sidecars的配置数量可以限定为必要的依赖。

网格中的服务和配置被组织成一个或多个命名空间（例如，Kubernetes命名空间或CF 组织/空间）。命名空间中的工作负载对同一命名空间中的其他工作负载具有隐式依赖性。此外，要声明对其他命名空间中的工作负载的依赖性，必须在当前命名空间中指定ServiceDependency资源。每个命名空间必须只能有一个名为“default”的ServiceDependency资源。如果给定命名空间中存在多个ServiceDependency资源，则系统的行为是未定义的。ServiceDependency资源中指定的依赖关系集将用于计算命名空间中每个工作负载的sidecar配置。

注1：如果网格中的工作负载仅依赖于同一命名空间中的其他工作负载，请在网格全局配置map（在values.yaml中）中将`defaultServiceDependency.importMode`设置为`SAME_NAMESPACE`。

注2：为了便于对Sidecar配置进行增量修剪，网格的默认导入模式设置为`ALL_NAMESPACES`。换句话说，每个工作负载都能够达到每个其他工作负载。在命名空间中添加ServiceDependency资源将自动修剪该命名空间中的工作负载的配置。

