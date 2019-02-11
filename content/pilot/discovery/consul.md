---
date: 2019-02-02T09:00:00+08:00
title: Consul实现
weight: 415
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务注册与发现抽象模型的Consul实现"
---

pilot discovery 的 Consul 实现。

## Controller的定义

Consul实现非常简单，一个 consul 的 client，一个monitor用来监听。

```go
type Controller struct {
	client  *api.Client
	monitor Monitor
}
```

## Discovery的实现

Services() 很简单，用 consul 的 client 获取所有服务，然后做一次格式转换即可：

```go
func (c *Controller) Services() ([]*model.Service, error) {
	data, err := c.getServices()

	services := make([]*model.Service, 0, len(data))
	for name := range data {
		endpoints, err := c.getCatalogService(name, nil)
		services = append(services, convertService(endpoints))
	}

	return services, nil
}
```

convertService()方法的细节，可以稍后研究一下，主要是对照一下 consul 的服务模型和 Pilot 的服务模型。

GetService()方法也是如此，调用 consul client 的 getCatalogService()方法获取数据，然后转一下格式：

```go
func (c *Controller) GetService(hostname model.Hostname) (*model.Service, error) {
	name, err := parseHostname(hostname)
	endpoints, err := c.getCatalogService(name, nil)
	return convertService(endpoints), nil
}
```

InstancesByPort() 方法，也是类似：

```go
func (c *Controller) InstancesByPort(hostname model.Hostname, port int,
	labels model.LabelsCollection) ([]*model.ServiceInstance, error) {

	name, err := parseHostname(hostname)
	endpoints, err := c.getCatalogService(name, nil)

	instances := []*model.ServiceInstance{}
	for _, endpoint := range endpoints {
		instance := convertInstance(endpoint)
		if labels.HasSubsetOf(instance.Labels) && portMatch(instance, port) {
			instances = append(instances, instance)
		}
	}

	return instances, nil
}
```

GetProxyServiceInstances()方法就有点惨，consul client 没有提供现成的方法，因此不得不先获取全量的service，然后逐个比对IP地址：

```go
func (c *Controller) GetProxyServiceInstances(node *model.Proxy) ([]*model.ServiceInstance, error) {
    // 获取全量的service
	data, err := c.getServices()
	out := make([]*model.ServiceInstance, 0)
    // 对每个service
	for svcName := range data {
        // 都获取该服务的服务实例信息
		endpoints, err := c.getCatalogService(svcName, nil)

        // 然后对每一个服务实例
		for _, endpoint := range endpoints {
			addr := endpoint.ServiceAddress
			if len(node.IPAddresses) > 0 {
                // 都比对一下IP地址
				for _, ipAddress := range node.IPAddresses {
					if ipAddress == addr {
						out = append(out, convertInstance(endpoint))
						break
					}
				}
			}
		}
	}

	return out, nil
}
```

