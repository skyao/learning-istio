---
date: 2018-09-29T20:00:00+08:00
title: Galley
weight: 600
description : "介绍istio Galley模块"
---

Istio Galley模块

### 介绍

Galley是Istio的顶级组件，负责配置摄取，处理和分发，是 Istio 1.0 中新引入的组件。

Galley负责将其余的Istio组件与从底层平台获取用户配置的细节隔离开来:

- 它包含用于收集配置的Kubernetes CRD侦听器
- 用于分发配置的MCP协议服务器实现
- 以及用于Kubernetes API Server进行预摄取(pre-ingest)验证的验证Web挂钩。