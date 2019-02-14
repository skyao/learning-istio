---
date: 2019-02-02T09:00:00+08:00
title: 聚合
weight: 519
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务注册的聚合"
---

## 定义

Registry指定service registry相关接口的集合

```go
type Registry struct {
	Name      serviceregistry.ServiceRegistry
	ClusterID string
	model.Controller
	model.ServiceDiscovery
	model.ServiceAccounts
}
```

Controller在不同的service registry之间聚合数据并监视更改

```go
type Controller struct {
	registries []Registry
	storeLock  sync.RWMutex
}
```

## 实现

### services()

```go
func (c *Controller) Services() ([]*model.Service, error) {
	// smap is a map of hostname (string) to service, used to identify services that
	// are installed in multiple clusters.
	smap := make(map[model.Hostname]*model.Service)

	services := make([]*model.Service, 0)
	var errs error
	// Locking Registries list while walking it to prevent inconsistent results
	for _, r := range c.GetRegistries() {
		svcs, err := r.Services()
		if err != nil {
			errs = multierror.Append(errs, err)
			continue
		}
		// Race condition: multiple threads may call Services, and multiple services
		// may modify one of the service's cluster ID
		clusterAddressesMutex.Lock()
		if r.ClusterID == "" { // Should we instead check for registry name to be on safe side?
			// If the service is does not have a cluster ID (consul, ServiceEntries, CloudFoundry, etc.)
			// Do not bother checking for the cluster ID.
			// DO NOT ASSIGN CLUSTER ID to non-k8s registries. This will prevent service entries with multiple
			// VIPs or CIDR ranges in the address field
			services = append(services, svcs...)
		} else {
			// This is K8S typically
			for _, s := range svcs {
				sp, ok := smap[s.Hostname]
				if !ok {
					// First time we see a service. The result will have a single service per hostname
					// The first cluster will be listed first, so the services in the primary cluster
					// will be used for default settings. If a service appears in multiple clusters,
					// the order is less clear.
					sp = s
					smap[s.Hostname] = sp
					services = append(services, sp)
				}

				// If the registry has a cluster ID, keep track of the cluster and the
				// local address inside the cluster.
				if sp.ClusterVIPs == nil {
					sp.ClusterVIPs = make(map[string]string)
				}
				sp.Mutex.Lock()
				sp.ClusterVIPs[r.ClusterID] = s.Address
				sp.Mutex.Unlock()
			}
		}
		clusterAddressesMutex.Unlock()
	}
	return services, errs
}
```

