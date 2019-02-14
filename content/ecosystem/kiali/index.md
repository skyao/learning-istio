---
date: 2018-09-30T14:00:00+08:00
title: kiali
weight: 2020
description : "介绍istio的UI管理界面kiali"
---

![](images/logo-header-light.svg)

## Kiali介绍

kiali关注服务网格的可观察性，官方给出的定位是：

>Kiali provides answers to the question: What are the microservices in my Istio service mesh doing ?
>
>Kiali为此问题提供答案：我的Istio服务网格中的微服务在做什么？

下面是官方网站给出的项目描述：

> 微服务架构将单体结构分解成组合在一起的许多小块。已经出现了用于保护服务之间的通信的模式，例如容错（通过超时，重试，断路等），以及分布式跟踪以便能够看到调用的去向。服务网格现在可以在平台级别上提供这些服务，并使应用程序编写者从这些任务中解放出来。路由决策在网格级别完成。Kiali与Istio合作，可视化服务网格拓扑，熔断器或请求率等功能。Kiali还包括Jaeger Tracing，可以提供开箱即用的分布式跟踪功能。

### 特性列表

- Service graph representation：服务图形展示
- Distributed tracing：分布式追踪
- Metrics collection and graphs：指标收集和图形
- Configuration validation：配置验证
- Health computation/display：健康计算/显示
- Service discovery：服务发现

## Kiali资料

### 官方信息

- [官网](https://www.kiali.io/)
- [kialiproject@twitter](https://twitter.com/kialiproject)
- [官方文档](https://www.kiali.io/documentation/)
- [kiali@github](https://github.com/kiali/kiali)
- [Kiali JIRA@jboss.org](https://docs.jboss.org/display/KIALI/Kiali+-+Service+Mesh+Observability)

### 博客文章

- [Observe what your Istio microservices mesh is doing with Kiali](https://developers.redhat.com/blog/2018/09/20/istio-mesh-visibility-with-kiali/): 2018年9月20日发布
- [阿里云Kubernetes Service Mesh实践进行时(7): 可观测性分析服务Kiali](https://yq.aliyun.com/articles/599921): 2018-06-06发布

### 视频资料

在youtube上有kiali的channel，[Kiali service mesh observability project](https://www.youtube.com/channel/UCcm2NzDN_UCZKk2yYmOpc5w/videos)，官方发布了一些介绍视频和demo视频

- [Kiali - Sprint #11 Demo - Service Mesh Observability](https://www.youtube.com/watch?v=MI2MipfKCsA): 2018年9月28日发布，这个demo比较新，推荐观看
- [Kiali Graph Overview September 20, 2018](https://www.youtube.com/watch?v=Jow5OFGbAac): 2018年9月21日发布，17分钟，这个视频比较新，推荐观看
- [Kiali Overview (around version 0.3)](https://www.youtube.com/watch?v=8HZlDGURzLc): 2018-05-15，七分钟
- [Kiali - Sprint #2 Demo](https://www.youtube.com/watch?v=5TbyY2y-bZ8): 2018年3月26日发布，24分钟，内容有点乱，从界面看这个版本的kiali也稍显粗糙，比5月份的视频在美观度上差很多。老的这些demo基本上没有看的价值，看最新的吧。