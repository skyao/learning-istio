---
date: 2018-09-29T20:00:00+08:00
title: 介绍
weight: 100
keywords:
- istio
description : "介绍istio的基本情况"
---

## 概述

Istio是一个开放平台，提供统一的方式来集成微服务，管理跨微服务的流量，执行策略和汇总遥测数据。Istio的控制面板在底层集群管理平台（如Kubernetes，Mesos等）上提供了一个抽象层。

Istio由以下组件组成：

- **Envoy** - 每微服务器的Sidecar代理，处理集群中的服务间和服务到外部服务的入口/出口流量。代理形成安全的微服务网格，提供丰富的功能，如服务发现，丰富的7层路由，熔断器，策略执行和遥测记录/报告功能。

	> 注意：服务网格不是overlay网络。它简化和增强了应用程序中的微服务通过底层平台提供的网络彼此通话的方式。

- **Mixer** - 由代理和微服务使用的中央组件，用于执行ACL，限流，配额，认证，请求跟踪和遥测收集等策略。

- **Pilot** - 负责在运行时配置代理的组件。

- **Galley** - 负责存储和分发Istio配置状态的组件。
Kubernetes changed how we deploy applications. Istio is going to change how we connect, manage, and secure them.
- **Broker** - 为基于Istio的服务实现Open Service Broker API的组件。

Istio目前只支持Kubernetes平台，我们计划在不久的将来支持其他平台，如Cloud Foundry和Mesos。

## 开发者

Istio是一个开源项目，有活跃的开发社区。该项目由Google和IBM的团队与Lyft的Envoy团队合作启动。

## 仓库

Istio项目分为多个GitHub库。每个存储库包含有关如何构建和测试它的信息。

- istio/API：该存储库定义了Istio平台的组件级API和通用配置格式。

- istio/istio： 存放各种Istio示例程序以及管理Istio开源项目的各种文档。

- istio/pilot： 该存储库包含用于填充抽象服务模型的平台特定代码，当应用程序拓扑更改时动态重新配置代理，以及将路由规则转换为代理特定配置。istioctl命令行工具也在此存储库中。

- istio/mixer： 该存储库包含用于为通过代理的流量执行各种策略的代码，并从代理和服务收集遥测数据。有与各种云平台接入的插件，策略管理服务和监控服务。

- istio/mixerclient: mixer API的客户端类库.

- istio/galley: 该存储库包含Istio配置管理和分发系统的代码。

- istio/broker： 该存储库包含Istio实现Open Service Broker API的代码。

- istio/proxy： Istio代理包含Envoy代理（以Envoy过滤器的形式）的扩展，允许代理将策略执行决策委托给混合器。

## 特性

**连接，管理和保护微服务的开放平台**

- 智能路由和负载均衡

	通过动态路由配置控制服务之间的流量，进行A/B测试，金丝雀发布，和使用红/黑部署逐步升级版本。

- 跨语言和平台的弹性

	通过在不利条件下将应用程序从网络分片和级联故障中屏蔽，从而提高可靠性。

- Fleet-Wide(舰队范围)的政策执行

	将组织策略应用于服务之间的互动，确保访问策略得到执行，资源在消费者之间公平分配。

- 深度遥测和报告

	了解服务之间的依赖关系，它们之间的流量的性质和流程，并使用分布式跟踪快速识别问题。

