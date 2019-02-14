---
date: 2019-02-07T18:00:00+08:00
title: 服务发现接口
weight: 512
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务发现"
---

## 接口定义

Pilot提供 ServiceDiscovery 服务用于获取Istio服务实例，源代码在 `istio/pilot/pkg/model/service.go` 下:

```go
type ServiceDiscovery interface {
    // 列举出当前系统中所有的服务
	Services() ([]*Service, error)
    // 通过hostname 获取服务，注意这个方法已经被废弃
	GetService(hostname Hostname) (*Service, error)
    // 通过给定的 hostname，service port 获取服务实例，并匹配任意给定的label。如果label为空则匹配所有实例。
	InstancesByPort(hostname Hostname, servicePort int, labels LabelsCollection) ([]*ServiceInstance, error)
    // 获取代理的服务实例
	GetProxyServiceInstances(*Proxy) ([]*ServiceInstance, error)
    // 获取代理位置
	GetProxyLocality(*Proxy) string
    // 获取管理端口
	ManagementPorts(addr string) PortList
    // 获取工作负载的健康检查信息
	WorkloadHealthCheckInfo(addr string) ProbeList
}
```

## 方法详解

### Services()

Services()方法列出系统中所有服务的声明。

```go
Services() ([]*Service, error)
```

只有服务定义，没有服务实例的数据。

### GetService()

GetService()方法通过 hostname 获取服务，注意这个方法已经被废弃：

```go
GetService(hostname Hostname) (*Service, error)
```

但是，请注意，在这个方法被废弃之后， ServiceDiscovery 接口并没有提供替代品，导致现在 ServiceDiscovery 接口中就没有方法可以查询 Service，只能通过 Services() 方法列出所有服务。

> 备注：不确认这个设计的由来，如果系统中服务数量不是太多的话，通过 Services() 方法取全量数据，然后自行获取单个服务也是很容易实现的。

### InstancesByPort()

InstancesByPort()方法用于获取服务实例列表，需要匹配 hostname，service port 和 标签列表。Label列表为空则表示匹配所有实例。

```go
InstancesByPort(hostname Hostname, servicePort int, labels LabelsCollection) ([]*ServiceInstance, error)
```

举例，假设要查询 catalog.mystore.com 的服务实例， InstancesByPort(catalog.myservice.com, 80) 会有如下 NetworkEndpoints 结果：

- NetworkEndpoint(172.16.0.1:8888), Service(catalog.myservice.com), Labels(foo=bar)
- NetworkEndpoint(172.16.0.2:8888), Service(catalog.myservice.com), Labels(foo=bar)
- NetworkEndpoint(172.16.0.3:8888), Service(catalog.myservice.com), Labels(kitty=cat)
- NetworkEndpoint(172.16.0.4:8888),  Service(catalog.myservice.com), Labels(kitty=cat)

使用特定Label调用InstancesByPort()方法会返回剪切后的子集, 如 InstancesByPort(catalog.myservice.com, 80, foo=bar) 会返回：

- NetworkEndpoint(172.16.0.1:8888), Service(catalog.myservice.com), Labels(foo=bar)
- NetworkEndpoint(172.16.0.2:8888), Service(catalog.myservice.com), Labels(foo=bar)

类似的概念适用于使用特定的端口，主机名和标签来调用此函数。

这个方法在 Istio 0.8 版本中引入，每次只能使用一个端口来调用。

特别说明：

- CDS (clusters.go) 调用这个方法来构建 'dnslb' 类型的集群
- EDS 调用这个方法来构建端点结果
- 在使用这个方法做其他任何事情 (除了调试和工具外)之前请参阅 istio-dev

### GetProxyServiceInstances()

GetProxyServiceInstances()方法返回与给定代理共处一地的服务实例。

```go
GetProxyServiceInstances(*Proxy) ([]*ServiceInstance, error)
```
共址通常意味着在相同的网络命名空间和安全上下文中运行。

作为Sidecar运行的代理将返回非空片。 独立代理将返回空切片。

这有两个原因导致它返回多个ServiceInstances而不是一个：

   -  ServiceInstance有一个NetworkEndpoint，它有一个端口。 但是服务可能有很多端口。 因此，实现此类服务的工作负载需要多个ServiceInstances，每个端口一个。（备注：这是上一节总结中提到的 ServiceInstance 模型和 ServicePort 一对一绑定）
   -  单个工作负载可以实现多个逻辑服务。（备注：这有悖于微服务的理念，不过理论上也还是可能存在的，但是Istio真的支持多个逻辑服务存在于一个服务实例中吗？）

在第二种情况下，可以通过相同的物理端口号实现多个服务，尽管每个服务具有不同的 ServicePort 和 NetworkEndpoint。 如果这些重叠服务中的任何一个不是基于HTTP或HTTP/2的，则行为是未定义的，因为如果请求没有 Host header，则监听器可能无法确定连接的预期目标。

### GetProxyLocality()

GetProxyLocality()方法返回代理运行的位置。

```go
GetProxyLocality(*Proxy) string
```

### ManagementPorts()

ManagementPorts() 方法列出与IPv4地址关联的一组管理端口。

```go
ManagementPorts(addr string) PortList
```

  这些管理端口通常由平台用于带外(out of band)管理任务，例如健康检查等。在代理以透明模式运行的情况下（捕获与服务实例IP地址之间的所有流量），为代理生成的配置不会操纵发往管理端口的流量。

### WorkloadHealthCheckInfo()

WorkloadHealthCheckInfo()方法列出与IPv4地址关联的探针列表。平台使用这些探针来识别执行运行状况检查的请求。

```go
WorkloadHealthCheckInfo(addr string) ProbeList
```



## 总结

ServiceDiscovery 接口的设计和通常的服务发现接口设计风格迥异，从使用者的角度看，基本只有两个主要方法可用：

- Services()方法列出系统中所有服务
- InstancesByPort()方法用于获取服务实例列表，需要给出 hostname，service port 和 Label列表

明显是一个高度订制的服务发现接口，而不是通用接口。