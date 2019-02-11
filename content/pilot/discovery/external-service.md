---
date: 2019-02-02T09:00:00+08:00
title: 外部服务
weight: 416
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务注册与发现抽象模型的外部服务"
---

Pilot Discovery 的 外部服务实现。

外部服务指 Istio ServiceEntry CRD 描述的服务信息，这些服务不在 k8s 管理范围内，因此它们的信息无法直接从 API server 的 service 信息中获取，需要从 ServiceEntry CRD 中读取。

## 定义

```go
type ServiceEntryStore struct {
	serviceHandlers  []serviceHandler
	instanceHandlers []instanceHandler
	store            model.IstioConfigStore

	storeMutex sync.RWMutex

	ip2instance map[string][]*model.ServiceInstance
	// Endpoints table. Key is the fqdn of the service, ':', port
	instances map[string][]*model.ServiceInstance

	changeMutex  sync.RWMutex
	lastChange   time.Time
	updateNeeded bool
}
```

## 实现

### Services()

```go
func (d *ServiceEntryStore) Services() ([]*model.Service, error) {
	services := make([]*model.Service, 0)
    // 拿到所有的 ServiceEntries 的config
	for _, config := range d.store.ServiceEntries() {
		services = append(services, convertServices(config)...)
	}

	return services, nil
}
```

### InstancesByPort()

```go
func (d *ServiceEntryStore) InstancesByPort(hostname model.Hostname, port int,
	labels model.LabelsCollection) ([]*model.ServiceInstance, error) {
	d.update()

	d.storeMutex.RLock()
	defer d.storeMutex.RUnlock()
	out := []*model.ServiceInstance{}

	instances, found := d.instances[string(hostname)]
	if found {
		for _, instance := range instances {
			if instance.Service.Hostname == hostname &&
				labels.HasSubsetOf(instance.Labels) &&
				portMatchSingle(instance, port) {
				out = append(out, instance)
			}
		}
	}

	return out, nil
}
```

