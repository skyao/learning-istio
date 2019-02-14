---
date: 2019-02-02T09:00:00+08:00
title: Pilot Discovery
weight: 510
description : "介绍istio的Pilot Discovery"
---

Pilot Discovery

源代码在 `istio/pilot/serviceregistry` 目录下。

## 平台支持的变迁

serviceregistry 目录下的 platform.go 文件的内容，代表了过去一年间，istio对服务注册平台支持的变迁。

2017-8-26，文件内容如下(去除不必要的内容和注释)，支持的平台有 Kubernetes 和 consul：

```go
const (
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
)
```

2017-9-17，增加了对Eureka的支持：

```go
const (
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
	EurekaRegistry ServiceRegistry = "Eureka"
)
```

2017-12-16，引入Cloud Fountry 平台:

```go
const (
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
	EurekaRegistry ServiceRegistry = "Eureka"
    CloudFoundryRegistry ServiceRegistry = "CloudFoundry"
)
```

2018-03-29，为了方便测试，引入了Mock，hard code了两个测试服务:

```go
const (
    // MockRegistry is a service registry that contains 2 hard-coded test services
    MockRegistry ServiceRegistry = "Mock"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
	EurekaRegistry ServiceRegistry = "Eureka"
    CloudFoundryRegistry ServiceRegistry = "CloudFoundry"
)
```

2018-6-7，增加了config-backed的service registry，服务注册信息来自ConfigStore:

```go
const (
    MockRegistry ServiceRegistry = "Mock"
    // ConfigRegistry is a service registry that listens for service entries in a backing ConfigStore
	ConfigRegistry ServiceRegistry = "Config"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
	EurekaRegistry ServiceRegistry = "Eureka"
    CloudFoundryRegistry ServiceRegistry = "CloudFoundry"
)
```

2018-7-12，删除了 Eureka 的支持(Eureka在2018年6月底宣布闭源)：

```go
const (
    MockRegistry ServiceRegistry = "Mock"
	ConfigRegistry ServiceRegistry = "Config"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
    CloudFoundryRegistry ServiceRegistry = "CloudFoundry"
)
```

2018-9-29，删除了 CloudFoundry (但并不是说不再支持CloudFoundry)，然后增加了MCP的支持：

```go
const (
    MockRegistry ServiceRegistry = "Mock"
	ConfigRegistry ServiceRegistry = "Config"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
    // MCPRegistry is a service registry backed by MCP ServiceEntries
	MCPRegistry ServiceRegistry = "MCP"
)
```

2019-1-3，以"remove misleading usless registries"的名义删除产生误导和无用的注册实现，Config和MCP被干掉：

```bash
const (
    MockRegistry ServiceRegistry = "Mock"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
)
```

2019-1-14，终于发现MCP被误删了，又加了回来：

```bash
const (
    MockRegistry ServiceRegistry = "Mock"
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	ConsulRegistry ServiceRegistry = "Consul"
	MCPRegistry ServiceRegistry = "MCP"
)
```

