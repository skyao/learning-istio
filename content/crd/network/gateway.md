---
date: 2018-09-29T20:00:00+08:00
title: Gateway
weight: 2013
menu:
  main:
    parent: "crd-network"
description : "Gateway CRD"
---

## 介绍

> 来自官方文档：https://preliminary.istio.io/zh/docs/reference/config/istio.networking.v1alpha3/#gateway

`Gateway` 描述了一个负载均衡器，用于承载网格边缘的进入和发出连接。这一规范中描述了一系列开放端口，以及这些端口所使用的协议、负载均衡的 SNI 配置等内容。



| 字段       | 类型                                                         | 描述                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `servers`  | `Server`| 必要字段。`Server` 定义列表。                                |
| `selector` | `map<string, string>`                                        | 必要字段。用一个或多个标签来选择一组 Pod 或虚拟机，用于应用 `Gateway` 配置。标签选择的范围是平台相关的。例如在 Kubernetes 上，选择范围包含所有可达的命名空间。 |

