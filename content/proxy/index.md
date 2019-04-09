---
date: 2018-09-29T21:00:00+08:00
title: Proxy
weight: 800
description : "介绍Istio中的envoy proxy"
---

Istio Proxy 项目包含对 Envoy 代码的扩展，这些扩展是通过 Envoy Filter 的方式提供的。

Istio Proxy有两个功能模块：

1. Envoy：这是 Envoy 本身，Istio Proxy 就是一个扩展了的 Envoy Proxy
2. Mixer Client：基于Envoy的扩展，用于Precondition/策略增强/遥测，和 Istio Mixer 模块交互。


