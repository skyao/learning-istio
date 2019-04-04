---
date: 2018-09-29T20:00:00+08:00
title: 架构
weight: 101
menu:
  main:
    parent: "introduction"
description : "Istio的架构"
---

### 整体架构

istio 1.1 版本官方文档中给出的 Istio 架构图:

![](images/arch.svg)

### 控制平面组件

kubernetes 部署下的控制平面组件如下：

```bash
$ kubectl -n istio-system get pod
NAME                                          READY     STATUS 
grafana-5f54556df5-s4xr4                      1/1       Running
istio-citadel-775c6cfd6b-8h5gt                1/1       Running
istio-galley-675d75c954-kjcsg                 1/1       Running
istio-ingressgateway-6f7b477cdd-d8zpv         1/1       Running
istio-pilot-7dfdb48fd8-92xgt                  2/2       Running
istio-policy-544967d75b-p6qkk                 2/2       Running
istio-sidecar-injector-5f7894f54f-w7f9v       1/1       Running
istio-telemetry-777876dc5d-msclx              2/2       Running
istio-tracing-5fbc94c494-558fp                1/1       Running
kiali-7c6f4c9874-vzb4t                        1/1       Running
prometheus-66b7689b97-w9glt                   1/1       Running
```

istio系统组件细化到进程级别:

![](images/process.jpg)

注意：galley 和 citadel 没有sidecar。

### 参考文档

- [Istio 庖丁解牛一：组件概览](http://www.servicemesher.com/blog/istio-analysis-1)
- [What is Istio?](https://istio.io/docs/concepts/what-is-istio/) @ istio docs