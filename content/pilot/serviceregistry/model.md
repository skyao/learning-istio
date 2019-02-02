---
date: 2019-02-02T09:00:00+08:00
title: 抽象模型
weight: 411
menu:
  main:
    parent: "pilot-serviceregistry"
description : "Pilot 服务注册与发现的抽象模型"
---


istio中，定义了一套服务注册的抽象模型，具体在文件｀istio/pilot/pkg/model/service.go｀中定义。

```go
// 这个文件描述Istio中服务的抽象模型(以及它们的实例)。
// 这个模型独立于底层平台 (Kubernetes, Mesos, 等.). 
// 平台特定适配器用在平台中找到的元数据填充模型对象的各个字段。
// 平台无关的proxy代码使用这个模型的表示来为7层代理sidecar生成配置文件。
// proxy代码是各个代理实现特有的。
```



## Service定义

```go
// Service描述Istio服务(如，catalog.mystore.com:8080)
// 每个服务有全限定域名(fully qualified domain name／FQDN)和一个或者多个端口
// 端口是服务用来监听连接的。可选的，服务可以关连有单个负载均衡器（load balancer）/虚拟IP（virtual IP）
// 这样FQDN的DNS查询解析到虚拟IP地址(负载均衡器IP)
//
// 例如，在kubernetes中，服务foo和主机名foo.default.svc.cluster.local关连，有一个虚拟IP10.0.1.1，
// 并且在端口80，8080上监听。
type Service struct {
    // 服务的Hostname, 例如"catalog.mystore.com"
	Hostname string `json:"hostname"`

	// Address指定服务的负载均衡器的IPv4地址
	Address string `json:"address,omitempty"`

	// Ports 是服务监听连接所在的网络端口集
	Ports PortList `json:"ports,omitempty"`

	// ExternalName 仅用于外部服务（external services），持有外部服务的DNS名称
	// External services 基于名称的解决方案，用来将外部服务实例表述为集群内的服务
    // 弃用：已过时，被MeshExternal和Resolution标志取代
	ExternalName string `json:"external"`

	// ServiceAccounts 指定运行服务的服务账号（service account）.
	ServiceAccounts []string `json:"serviceaccounts,omitempty"`

	// MeshExternal (如果为true) 表明这个服务是在mesh之外。
	// 这些服务通过Istio的 ExternalService 规范来定义.
	MeshExternal bool

	// LoadBalancingDisabled 表明这个服务不会有负载均衡
	// 弃用 : 已过时，被MeshExternal和Resolution标志取代
	LoadBalancingDisabled bool `json:"-"`

	// Resolution 指出在路由流量前服务实例需要如何被解析。
	// 服务注册中的大多数服务将使用静态负载均衡，proxy将决定接收流量的服务实例。
	// 外部服务可以使用DNS负载均衡 (例如 proxy 将查询DNS服务器来获取服务的IP地址)，
	// 也可以使用透传模式（passthrough model） (例如 proxy 将转发流量到调用者要求的网络端口)
	Resolution Resolution
}
```



### PortList

```go
// PortList 是一组port
type PortList []*Port

// Port 代表网络端口，服务在这里监听连接。
// 端口和使用这个端口的协议类型相关联。
type Port struct {
	// Name 是端口对象的可读名称。
	// 当服务有多个端口时，name字段是必须的
	Name string `json:"name,omitempty"`

	// Port 是可以访问到服务可以的端口号. Does not necessarily
	// 不是非要映射到服务后面的实例的对应端口
	// 见下面的 NetworkEndpoint 定义.
	Port int `json:"port"`

	// Protocol 是用于这个端口的协议.
	Protocol Protocol `json:"protocol,omitempty"`

	// 结合mesh的AuthPolicy, 控制envoy到envoy的通讯的认证。
	// 这个值提取自服务的注解.
	AuthenticationPolicy meshconfig.AuthenticationPolicy `json:"authentication_policy"`
}
```

### Protocol

```go
// Protocol 定义用于端口的网络协议
type Protocol string

const (
	ProtocolGRPC Protocol = "GRPC"
	ProtocolHTTPS Protocol = "HTTPS"
	ProtocolHTTP2 Protocol = "HTTP2"
	// HTTP/1.1 traffic，注意HTTP/1.0 或者更早版本不被支持
	ProtocolHTTP Protocol = "HTTP"
	// TCP，这是默认协议
	ProtocolTCP Protocol = "TCP"
	// UDP.注意当前还不支持
	ProtocolUDP Protocol = "UDP"
	ProtocolMongo Protocol = "Mongo"
	ProtocolRedis Protocol = "Redis"
	ProtocolUnsupported Protocol = "UnsupportedProtocol"
)
```

###meshconfig.AuthenticationPolicy

meshconfig.AuthenticationPolicy的定义在istio/api仓库中，路径为`istio/api/mesh/v1alpha1/config.pb.go`。

```go
// AuthenticationPolicy 定义认证策略。
// 可以为不同的范围(mesh, service …)设置, 并且带有非INHERIT值的最窄范围将被使用
// Mesh 策略不能是 INHERIT.
type AuthenticationPolicy int32

const (
	// 不要加密envoy到envoy的流量.
	AuthenticationPolicy_NONE AuthenticationPolicy = 0
	// Envoy到Envoy的流量被包装为mutual TLS 连接.
	AuthenticationPolicy_MUTUAL_TLS AuthenticationPolicy = 1
	// 使用父范围定义的策略。不能用于mesh策略。
	AuthenticationPolicy_INHERIT AuthenticationPolicy = 1000
)
```

### Resolution

```go
// Resolution 指出在路由流量前服务实例需要如何被解析。
type Resolution int

const (
	// ClientSideLB  意味着proxy将从它的本地负载均衡池中决定端点。
	ClientSideLB Resolution = iota
	// DNSLB 意味着proxy将解析DNS地址并转发到解析后的地址
	DNSLB
	// Passthrough 意味着proxy将转发流量到调用者要求的目标IP
	Passthrough
)
```

Service模型的Json表述如下：

```json
{
    hostname: "example-service1.default",
    address: "10.0.1.1",
    ports: {
        [{
            name: "http",
            port: 80,
            protocol: "HTTP",
            authentication_policy: 0
        },{
            name: "grpc",
            port: 8080,
            protocol: "GRPC",
            authentication_policy: 0
        }]
    },
    serviceaccounts: ["accout1", "account2"],
    meshExternal: false,
    resolution: 0
}
```

## ServiceInstance

```go
// ServiceInstance 表示服务特定版本的单个实例。
// 它绑定网络端点(ip:port),服务描述(适用于不同版本)和标签集，标签集描述和这个实例关联的服务版本。
//
// 因为ServiceInstance有单个的NetworkEndpoint, 而NetworkEndpoint有单个端口, 
// 所有要表示在多个端口上监听的工作需要多个ServiceInstances。
//
// 和服务实例关联的label在每个网络端口中是唯一的。
// 每个服务实例网络端点有一套定义良好的标签集合。
//
// 例如, 和catalog.mystore.com关联的服务实例集合建模如下：
//      --> NetworkEndpoint(172.16.0.1:8888), Service(catalog.myservice.com), Labels(foo=bar)
//      --> NetworkEndpoint(172.16.0.2:8888), Service(catalog.myservice.com), Labels(foo=bar)
//      --> NetworkEndpoint(172.16.0.3:8888), Service(catalog.myservice.com), Labels(kitty=cat)
//      --> NetworkEndpoint(172.16.0.4:8888), Service(catalog.myservice.com), Labels(kitty=cat)
type ServiceInstance struct {
	Endpoint         NetworkEndpoint `json:"endpoint,omitempty"`
	Service          *Service        `json:"service,omitempty"`
	Labels           Labels          `json:"labels,omitempty"`
	AvailabilityZone string          `json:"az,omitempty"`
	ServiceAccount   string          `json:"serviceaccount,omitempty"`
}
```

### NetworkEndpoint

```go
// NetworkEndpoint 定义和服务的实例关联的网络地址(IP:port)
// 服务有一个或者多个实例，每个实例运行于容器／虚拟机／pod。
// 如果一个服务有多个端口，那么同一个实例IP会期待于在多个端口上监听（每个服务端口一个）
// 注意和实例关联的端口不是一定要和服务关联的端口一样。
// 取决于网络搭建(NAT, overlays), 可以不同.
//
// 例如, 如果 catalog.mystore.com 可以通过端口80和8080访问,
// 并且它影射到IP为172.16.0.1的实例, 然后端口80的连接将转发到端口55446，
// 而端口8080的连接将转发到端口33333。
//
// 那么在内部，我们就有服务catalog.mystore.com的两个端点结构：
//  --> 172.16.0.1:54546 (with ServicePort pointing to 80) 和
//  --> 172.16.0.1:33333 (with ServicePort pointing to 8080)
type NetworkEndpoint struct {
	// 网络端点的地址，通常是IPv4地址
	Address string

	// 这个实例监听连接的端口号。
	// 可以和服务访问的端口不一样。
	// 例如, catalog.mystore.com:8080 -> 172.16.0.1:55446
	Port int

	// 在服务声明中声明的端口。
	// 这是和当前实例关联的服务的端口(例如, catalog.mystore.com)
	ServicePort *Port
}
```

### Labels

```go
// Labels 是任意字符串的非空集合。
// 服务的每个版本可以通过和这个版本关联的标签的独特集合来区分。 
// 这些标签被分配到特定服务版本的所有实例。
// 例如, 假定 catalog.mystore.com 有两个版本 v1 和 v2. v1 实例
// 有标签 gitCommit=aeiou234, region=us-east, 而 v2 实例有
// 标签 name=kittyCat,region=us-east.
type Labels map[string]string
```

服务实例的Json表示：

```son
{
	endpoint: {
        address: "172.16.0.1",
        port: 54546,
        servicePort: {
            name: "http",
            port: 80,
            protocol: "HTTP",
            authentication_policy: 0
        }
	},
    service: {
        hostname: "example-service1.default",
        address: "10.0.1.1",
        ports: {
            [{
                name: "http",
                port: 80,
                protocol: "HTTP",
                authentication_policy: 0
            },{
                name: "grpc",
                port: 8080,
                protocol: "GRPC",
                authentication_policy: 0
            }]
        },
        serviceaccounts: ["accout1", "account2"],
        meshExternal: false,
        resolution: 0
    },
    lebels: {
		gitCommit: "aeiou234", 
		region: "us-east"
    },
    az: "zone1",
    serviceaccount: "account1"
｝
```



