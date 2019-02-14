---
date: 2018-09-29T20:00:00+08:00
title: Istio Network CRD
weight: 410
description : "介绍istio的network CRD"
---


networking.istio.io

| CRD               | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| VirtualService    | VirtualService 定义了一系列针对指定服务的流量路由规则。每个路由规则都针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务（或者目标服务的子集或版本）。 |
| DestinationRule   | `DestinationRule` 所定义的策略，决定了经过路由处理之后的流量的访问策略。这些策略中可以定义负载均衡配置、连接池尺寸以及外部检测（用于在负载均衡池中对不健康主机进行识别和驱逐）配置。 |
| Gateway           | `Gateway` 描述了一个负载均衡器，用于承载网格边缘的进入和发出连接。这一规范中描述了一系列开放端口，以及这些端口所使用的协议、负载均衡的 SNI 配置等内容。 |
| ServiceEntry      | `ServiceEntry` 能够在 Istio 内部的服务注册表中加入额外的条目，从而让网格中自动发现的服务能够访问和路由到这些手工加入的服务。 |
| EnvoyFilter       | EnvoyFilter 对象描述了针对代理服务的过滤器，这些过滤器可以定制由 Istio Pilot 生成的代理配置。 |
| ServiceDependency | `ServiceDependency`描述工作负载依赖的服务集合。默认情况下，Istio建立的服务网格将具有完整的网格连接 - 即每个工作负载都拥有访问网格中的每个其他工作负载的代理配置。但是，大多数连接图在实践中都很稀疏。ServiceDependency提供了一种的方法来声明与每个工作负载关联的服务依赖关系，以便发送到sidecars的配置数量可以限定为必要的依赖。 |

