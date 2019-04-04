---
date: 2018-09-29T20:00:00+08:00
title: 代码仓库
weight: 102
menu:
  main:
    parent: "introduction"
description : "Istio的代码仓库"
---

### 概况

istio 项目在github的组织为 https://github.com/istio ，这里有istio的几个主要代码仓库：

- api：API/配置/标准词汇表等的定义
- istio: 主要代码仓库，包含istio控制平面的所有组件: pilot, mixer, citadel, galley
- proxy：Istio的Sidecar，在envoy的基础上扩展了mixer client
- istio.io：istio.io 网站的源文件，包括文档
- community：Istio社区管理的各种资料

### istio/istio仓库

https://github.com/istio/istio 打包出来的镜像和命令:

| 容器名                   | 镜像名                 | 启动命令          | 源码入口                          |
| :----------------------- | :--------------------- | :---------------- | :-------------------------------- |
| Istio_init               | istio/proxy_init       | istio-iptables.sh | istio/tools/deb/istio-iptables.sh |
| istio-proxy              | istio/proxyv2          | pilot-agent       | istio/pilot/cmd/pilot-agent       |
| sidecar-injector-webhook | istio/sidecar_injector | sidecar-injector  | istio/pilot/cmd/sidecar-injector  |
| discovery                | istio/pilot            | pilot-discovery   | istio/pilot/cmd/pilot-discovery   |
| galley                   | istio/galley           | galley            | istio/galley/cmd/galley           |
| mixer                    | istio/mixer            | mixs              | istio/mixer/cmd/mixs              |
| citadel                  | istio/citadel          | istio_ca          | istio/security/cmd/istio_ca       |

另外还有2个其他命令:

| 命令       | 源码入口                      | 作用                                                         |
| :--------- | :---------------------------- | :----------------------------------------------------------- |
| mixc       | istio/mixer/cmd/mixc          | 用于和Mixer server 交互的客户端                              |
| node_agent | istio/security/cmd/node_agent | 用于node上安装安全代理, 这在Mesh Expansion特性中会用到, 即k8s和vm打通. |

### istio/proxy

<https://github.com/istio/proxy> 该项目会打包为 Envoy 二进制程序, 该二进制程序会被打包为 istio 的 Sidecar 容器镜像`istio/proxyv2`中.

### 参考文档

- [Istio 庖丁解牛一：组件概览](http://www.servicemesher.com/blog/istio-analysis-1)