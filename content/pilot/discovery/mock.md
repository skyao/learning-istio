---
date: 2019-02-02T09:00:00+08:00
title: Mock实现
weight: 413
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot Service Discovery的Mock实现"
---

Service Discovery的Mock实现，代码在：

`istio/pilot/pkg/serviceregistry/memory/discovery.go`

### NewDiscovery()

NewDiscovery()方法构建一个基于内存的ServiceDiscovery

```go
func NewDiscovery(services map[model.Hostname]*model.Service, versions int) *ServiceDiscovery {
	return &ServiceDiscovery{
		services: services,
		versions: versions,
	}
}
```

### ServiceDiscovery接口

ServiceDiscovery 是一个基于内存的discovery interface，实现了 model.ServiceDiscovery：

```go
type ServiceDiscovery struct {
    // 保存用于测试的服务信息
	services                      map[model.Hostname]*model.Service
	versions                      int
    // 保存期望获得的代理服务实例
	WantGetProxyServiceInstances  []*model.ServiceInstance
    // 各种预设的error，用于模拟出错的情况
	ServicesError                 error
	GetServiceError               error
	InstancesError                error
	GetProxyServiceInstancesError error
}
```

这个mock registry是用于测试目的。

先看看和服务发现直接相关的几个方法的实现：

```go
// AddService 用来添加服务到registry
func (sd *ServiceDiscovery) AddService(name model.Hostname, svc *model.Service) {
    // 保存在 services 字段中
	sd.services[name] = svc
}

// Services 方法实现 discovery 接口，返回所有服务
func (sd *ServiceDiscovery) Services() ([]*model.Service, error) {
    // 看看是否有预设的 ServicesError 错误
	if sd.ServicesError != nil {
		return nil, sd.ServicesError
	}
	out := make([]*model.Service, 0, len(sd.services))
    // 简单的将 services 字段的内容返回
	for _, service := range sd.services {
		out = append(out, service)
	}
	return out, sd.ServicesError
}

// GetService 方法实现 discovery 接口，返回单个服务
func (sd *ServiceDiscovery) GetService(hostname model.Hostname) (*model.Service, error) {
    // 看看是否有预设的 ServicesError 错误
	if sd.GetServiceError != nil {
		return nil, sd.GetServiceError
	}
    // 以hostname为key简单查询，返回值有可能为nil
	val := sd.services[hostname]
	return val, sd.GetServiceError
}

// InstancesByPort 方法实现 discovery 接口，返回服务实例
func (sd *ServiceDiscovery) InstancesByPort(hostname model.Hostname, num int,
	labels model.LabelsCollection) ([]*model.ServiceInstance, error) {
    // 看看是否有预设的 InstancesError 错误
	if sd.InstancesError != nil {
		return nil, sd.InstancesError
	}
    // 先检查有没有服务
	service, ok := sd.services[hostname]
	if !ok {
		return nil, sd.InstancesError
	}
	out := make([]*model.ServiceInstance, 0)
    // 如果服务类型是 external，则返回空列表
	if service.External() {
		return out, sd.InstancesError
	}
    // 按给定的端口在服务的 ServicePort 列表中查询
	if port, ok := service.Ports.GetByPort(num); ok {
        // 构建多个不同版本的的实例，版本数量有字段 versions 决定
		for v := 0; v < sd.versions; v++ {
			if labels.HasSubsetOf(map[string]string{"version": fmt.Sprintf("v%d", v)}) {
				out = append(out, MakeInstance(service, port, v, "zone/region"))
			}
		}
	}
	return out, sd.InstancesError
}
```

和 Proxy 相关的方法实现：

```go
// GetProxyServiceInstances 实现 discovery interface
func (sd *ServiceDiscovery) GetProxyServiceInstances(node *model.Proxy) ([]*model.ServiceInstance, error) {
    // 看看是否有预设的错误
	if sd.GetProxyServiceInstancesError != nil {
		return nil, sd.GetProxyServiceInstancesError
	}
	if sd.WantGetProxyServiceInstances != nil {
		return sd.WantGetProxyServiceInstances, nil
	}
	out := make([]*model.ServiceInstance, 0)
    // 游历所有保存的服务
	for _, service := range sd.services {
		if !service.External() {
			for v := 0; v < sd.versions; v++ {
				// Only one IP for memory discovery?
                // makeIP方法返回的IP地址要和node的IP匹配
				if node.IPAddresses[0] == MakeIP(service, v) {
					for _, port := range service.Ports {
						out = append(out, MakeInstance(service, port, v, "zone/region"))
					}
				}
			}

		}
	}
	return out, sd.GetProxyServiceInstancesError
}

func (sd *ServiceDiscovery) GetProxyLocality(node *model.Proxy) string {
	// not implemented
	return ""
}
```

