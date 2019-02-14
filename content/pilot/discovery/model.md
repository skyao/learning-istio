---
date: 2019-02-02T09:00:00+08:00
title: 服务抽象模型
weight: 511
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务注册与发现的服务抽象模型"
---

## 介绍

istio中，定义了一套服务注册的抽象模型，具体在文件｀istio/pilot/pkg/model/service.go｀中定义。

说明如下：


> 文件描述Istio中服务的抽象模型(以及它们的实例)。这个模型独立于底层平台 (Kubernetes, Mesos, 等.). 平台特定适配器用在平台中找到的元数据填充模型对象的各个字段。
>
> 平台无关的proxy代码使用这个模型的表示来为7层代理sidecar生成配置文件。proxy代码是各个代理实现特有的。

结合Pilot的架构图，就很容易理解了：

![](images/PilotAdapters.svg)

这里所说的服务抽象模型，就是图中的"Abstract Model"。

## Service定义

Service描述Istio服务(如，catalog.mystore.com:8080)。每个服务有全限定域名(fully qualified domain name／FQDN)和一个或者多个端口。端口是服务用来监听连接的。可选的，服务可以关连有单个负载均衡器（load balancer）/虚拟IP（virtual IP），这样FQDN的DNS查询解析到虚拟IP地址(负载均衡器IP)。例如，在kubernetes中，服务foo和主机名foo.default.svc.cluster.local关连，有一个虚拟IP 10.0.1.1，并且在端口80，8080上监听。

```go
type Service struct {
    // 服务的Hostname, 例如"catalog.mystore.com"
	Hostname string `json:"hostname"`

	// Address 指定服务的负载均衡器的IPv4地址
	Address string `json:"address,omitempty"`
    
    // ClusterVIPs 指定服务所在的每个集群的负载均衡器的服务地址
	ClusterVIPs map[string]string `json:"cluster-vips,omitempty"`

	// Ports 是服务监听连接所在的网络端口集
	Ports PortList `json:"ports,omitempty"`

	// ServiceAccounts 指定运行服务的服务账号（service account）.
	ServiceAccounts []string `json:"serviceaccounts,omitempty"`

	// MeshExternal (如果为true) 表明这个服务是在mesh之外。
	// 这些服务通过 Istio 的 ExternalService 规范来定义.
	MeshExternal bool

	// Resolution 指出在路由流量前服务实例需要如何被解析。
	// 服务注册中的大多数服务将使用静态负载均衡，proxy将决定接收流量的服务实例。
	// 外部服务可以使用DNS负载均衡 (例如 proxy 将查询DNS服务器来获取服务的IP地址)，
	// 也可以使用透传模式（passthrough model） (例如 proxy 将转发流量到调用者要求的网络终端)
	Resolution Resolution

	// CreationTime 记录服务创建的时间，如果可用
	CreationTime time.Time `json:"creationTime,omitempty"`

	// Attributes 包含和服务相关的额外属性，主要被mixer和RBAC用于策略加强。
	Attributes ServiceAttributes
}
```

### Service字段定义

#### PortList和Port

```go
// PortList 是一组port
type PortList []*Port

// Port 代表网络端口，服务在这里监听连接。
// 端口和使用这个端口的协议类型相关联。
type Port struct {
	// Name 是端口对象的可读名称。
	// 当服务有多个端口时，name字段是必须的
	Name string `json:"name,omitempty"`

	// Port 是可以访问到服务可以的端口号.
	// 不是非要映射到服务后面的实例的对应端口
	// 见下面的 NetworkEndpoint 定义.
	Port int `json:"port"`

	// Protocol 是用于这个端口的协议.
	Protocol Protocol `json:"protocol,omitempty"`
}
```

#### Protocol

```go
// Protocol 定义用于端口的网络协议
type Protocol string

const (
	ProtocolGRPC Protocol = "GRPC"
    ProtocolGRPCWeb Protocol = "GRPC-Web"
    // HTTP/1.1 traffic，注意HTTP/1.0 或者更早版本不被支持
	ProtocolHTTP Protocol = "HTTP"
	ProtocolHTTPS Protocol = "HTTPS"
	ProtocolHTTP2 Protocol = "HTTP2"
	// TCP，这是默认协议
	ProtocolTCP Protocol = "TCP"
    ProtocolTLS Protocol = "TLS"
	// UDP.注意当前还不支持
	ProtocolUDP Protocol = "UDP"
	ProtocolMongo Protocol = "Mongo"
	ProtocolRedis Protocol = "Redis"
    ProtocolMySQL Protocol = "MySQL"
	ProtocolUnsupported Protocol = "UnsupportedProtocol"
)
```

#### Resolution

```go
// Resolution 指出在路由流量前服务实例需要如何被解析。
type Resolution int

const (
	// ClientSideLB  意味着proxy将从它的本地负载均衡池中选择终端。
	ClientSideLB Resolution = iota
	// DNSLB 意味着proxy将解析DNS地址并转发到解析后的地址
	DNSLB
	// Passthrough 意味着proxy将转发流量到调用者要求的目标IP
	Passthrough
)
```

#### ServiceAttributes

ServiceAttributes表示服务自定义属性组：

```go
type ServiceAttributes struct {
	// Name 是 "destination.service.name" 属性
	Name string
	// Namespace 是 "destination.service.namespace" 属性
	Namespace string
	// UID 是 "destination.service.uid" 属性
	UID string
	// ConfigScope 定义当namespace被导入时，namespace中的服务的可见性
	ConfigScope networking.ConfigScope
}
```

ConfigScope 定义当 namespace 被导入时，在 namespace 中 Istio 配置实体的可见性。默认所有配置实体都是 public 。当包含 private 配置的 namespace 被导入到 Sidecar时，private 的配置将不会导入。

```go
type ConfigScope int32

const (
	// 有这个范围的Config对于mesh中所有的工作负载都可见。
	ConfigScope_PUBLIC ConfigScope = 0
	// 有这个范围的Config 只对和配置资源在同一个namespace下的工作负载可见
	ConfigScope_PRIVATE ConfigScope = 1
)
```

### Service模型的Json表述

Service模型的Json表述如下：

```json
{
    hostname: "example-service1.default",
    address: "10.0.1.105",
    clusterVIPs: {
        "vip1": "172.100.1.10", 
        "vip2": "172.100.2.11"
    },
    ports: {
        [{
            name: "http",
            port: 80,
            protocol: "HTTP"
        },{
            name: "grpc",
            port: 8080,
            protocol: "GRPC"
        }]
    },
    serviceaccounts: ["accout1", "account2"],
    meshExternal: false,
    resolution: 0,
    creationTime: ,
    attributes: {
    	name: "example-service1",
    	namespace: "default",
    	uid: "1234",
    	configScope: 0
	}
}
```

## ServiceInstance定义

ServiceInstance 表示服务特定版本的单个实例。

它绑定网络端点(ip:port), 服务描述(适用于不同版本)和标签集，标签集描述和这个实例关联的服务版本。

因为 ServiceInstance 有单个的NetworkEndpoint, 而NetworkEndpoint有单个端口,  所有要表示在多个端口上监听的工作负载需要多个ServiceInstances。

和服务实例关联的label在每个网络端口中是唯一的。每个服务实例网络端点有一套定义良好的标签集合。

例如, 和catalog.mystore.com关联的服务实例集合建模如下：

- NetworkEndpoint(172.16.0.1:8888), Service(catalog.myservice.com), Labels(foo=bar)
- NtworkEndpoint(172.16.0.2:8888), Service(catalog.myservice.com), Labels(foo=bar)
- NetworkEndpoint(172.16.0.3:8888), Service(catalog.myservice.com), Labels(kitty=cat)
- NetworkEndpoint(172.16.0.4:8888), Service(catalog.myservice.com), Labels(kitty=cat)

```go
type ServiceInstance struct {
	Endpoint         NetworkEndpoint `json:"endpoint,omitempty"`
	Service          *Service        `json:"service,omitempty"`
	Labels           Labels          `json:"labels,omitempty"`
	ServiceAccount   string          `json:"serviceaccount,omitempty"`
}
```

### ServiceInstance字段定义

#### NetworkEndpoint

NetworkEndpoint 定义和服务的实例关联的网络地址(IP:port)。服务有一个或者多个实例，每个实例运行于容器/虚拟机/pod。如果一个服务有多个端口，那么同一个实例IP会期待于在多个端口上监听（每个服务端口一个）

注意和实例关联的端口不是一定要和服务关联的端口一样。取决于网络搭建(NAT, overlays), 可以不同。例如, 如果 catalog.mystore.com 可以通过端口80和8080访问, 并且它影射到IP为 172.16.0.1 的实例, 然后端口80的连接将转发到端口55446，而端口8080的连接将转发到端口33333。

那么在内部，我们就有 catalog.mystore.com 服务的两个端点结构：

- 172.16.0.1:54546 (带有指向80的ServicePort) 
- 172.16.0.1:33333 (带有指向8080的ServicePort)

```go
type NetworkEndpoint struct {
    // Family 指示端点的类型， 如TCP 或者 Unix Domain Socket
	Family AddressFamily
    
	// 网络端点的地址，通常是IPv4地址，如果Family是`AddressFamilyTCP`；如果Family是`AddressFamilyUnix`，则是到domain socket的地址
	Address string

	// 这个实例监听连接的端口号。
	// 可以和服务访问的端口不一样。
	// 例如, catalog.mystore.com:8080 -> 172.16.0.1:55446
    // `AddressFamilyUnix`忽略这个字段。
	Port int

	// 在服务声明中声明的端口。
	// 这是和当前实例关联的服务(例如, catalog.mystore.com)的端口
	ServicePort *Port
    
	// 定义平台特有工作负载实例标识符（可选）
	UID string

	// 端点所在的网络
	Network string

    // 端点所在的局域(locality)
	Locality string

	// 和这个端点关联的负载均衡权重
	LbWeight uint32
}
```

#### AddressFamily

```go
// AddressFamily 表示用于访问 NetworkEndpoint 的传输类型
type AddressFamily int

const (
	// AddressFamilyTCP 表示连接到TCP端点的地址。它由IP地址或者主机名和端口组成。
	AddressFamilyTCP AddressFamily = iota
	// AddressFamilyUnix 表示连接到Unix Domain Socket的地址。由socket 文件地址组成。
	AddressFamilyUnix
)
```

#### Labels

Labels 是任意字符串的非空集合(set)。

服务的每个版本可以通过和这个版本关联的标签的独特集合来区分。 这些标签被分配到特定服务版本的所有实例。例如, 假定 catalog.mystore.com 有两个版本 v1 和 v2:

- v1 实例有标签 gitCommit=aeiou234, region=us-east
- 而 v2 实例有标签 name=kittyCat,region=us-east

```go
type Labels map[string]string
```

服务实例的Json表示：

```json
{
	endpoint: {
		family: 0,
        address: "172.16.0.1",
        port: 54546,
        servicePort: {
            name: "http",
            port: 80,
            protocol: "HTTP",
        },
        uid:"",
        network:"",
        locality:""
        lwWeight: 10
	},
    service: {
        hostname: "example-service1.default",
        address: "10.0.1.105",
        clusterVIPs: {
            "vip1": "172.100.1.10", 
            "vip2": "172.100.2.11"
        },
        ports: {
            [{
                name: "http",
                port: 80,
                protocol: "HTTP"
            },{
                name: "grpc",
                port: 8080,
                protocol: "GRPC"
            }]
        },
        serviceaccounts: ["accout1", "account2"],
        meshExternal: false,
        resolution: 0,
        creationTime: ,
        attributes: {
            name: "example-service1",
            namespace: "default",
            uid: "1234",
            configScope: 0
        }
    },
    labels: {
		gitCommit: "aeiou234", 
		region: "us-east"
    },
    serviceaccount: "account1"
｝
```

## 服务模型总结

Istio定义了服务的抽象模型，包括 Service 和 ServiceInstance，和通常的服务注册发现系统定义的服务模型相比，不同的地方主要有：

### 服务标识

在Istio的模型中，服务的标识符是用 hostname，比如"catalog.mystore.com", 还有FQDN的支持要求。一般的服务注册机制是用一个简单的服务名如"UserService"来标识单个服务，可能会有service namespace或者service group用来避免重名。

虽然本质上没有大的差别，都是通过一个唯一标识来识别服务。

Istio的模型中对服务的标识是非常明确的：用 hostname 来标识，而且是FQDN。这样一方面符合 kubernetes 的习惯，另一方面也为DNS的使用留下了非常好的基础。

在Istio中定义 VirtualService 和 DestinationRule 时，我们会发现，对于这些规则要生效的服务，Istio 同样是通过 hostname 来标识的。

### ClusterVIP

在Istio的模型中，有 clusterVIPs 字段，指定服务在不同的集群中负载均衡器的地址。

TBD: 具体使用方式待细细研究。

### 服务端口

Istio的服务模型中，端口是遵循 12 factor 中的 Port Binding 原则，非常明确的将端口和协议绑定，每个服务对外提供访问的端口都是和具体的通讯协议如HTTP绑定。

然后，每个 serviceInstance 可以有自己的特别的 port，容许和 ServicePort 不同，当然会有关联关系。

但是，对于客户端使用者，访问应该都是通过 ServicePort 来进行。

### ServiceAccount

Istio的服务模型中，提供了service account信息，这位基于service account进行安全控制提供了基础。

### MeshExternal

MeshExternal 属性表示服务在mesh之外，也就是在当前Istio的服务注册机制之外，这为访问外部服务提供了一个可用的方式，也为打通不同的体系留下了伏笔。

### ConfigScope

可以通过ConfigScope属性来定义服务在namespace外的可见性，当设置为 private 时就只有当前 namespace 可见，这样可以实现让部分设计为 namespace 私有的服务不被外面访问。

也有一些不太容易适应的设计，比如：

### ServiceInstance和ServicePort的一对一绑定

因为 ServiceInstance 只有单个的NetworkEndpoint, 而NetworkEndpoint 只有单个端口, 所有要表示在多个端口上监听的工作负载需要多个ServiceInstances。

也即是说，如果一个 Service ，定义有三个 ServicePort，比如 HTTP 、 HTTPS 、GRPC，则这个服务的一个实际运行的实例（也就是一个进程）就需要三个 ServiceInstance 对象，分别对应这三个 ServicePort。

通常情况下，会比较自然的采用的设计是：一个实际运行的实例对应一个 ServiceInstance 对象，然后在这个 ServiceInstance 对象中保存三个 Service Port 对象。

Istio的这个设计，使得 ServicePort 在进行服务发现时显得特别的重要，在后面一节讲述 Istio 的服务发现接口时再仔细介绍。

## 服务模型补充

### IstioEndpoint

IstioEndpoint 具有关于特定服务和分片的单个地址+端口的信息。 

```go
// TODO: Replace NetworkEndpoint and ServiceInstance with Istio endpoints
type IstioEndpoint struct {
	Labels map[string]string  		// ServiceInstance
	Family AddressFamily			// NetworkEndpoint
	Address string					// NetworkEndpoint
	EndpointPort uint32				// NetworkEndpoint
	ServicePortName string			// NetworkEndpoint, 类型不同，从 int 变成了 string
	UID string						// NetworkEndpoint
	EnvoyEndpoint *endpoint.LbEndpoint // ServiceInstance
	ServiceAccount string 			// ServiceInstance
	Network string					// NetworkEndpoint
	Locality string					// NetworkEndpoint
	LbWeight uint32					// NetworkEndpoint
}
```

源代码中有TODO说要用 IstioEndpoint 替代 NetworkEndpoint 和 ServiceInstance。

IstioEndpoint 的特别说明：

-  ServicePortName 替代了 ServicePort，因为在进行端点回调时，端口号和协议可能不可用。
-  不再分成一个 ServiceInstance 和一个 NetworkEndpoint  - 两者都在一个结构中
-  没有指向Service的指针 - 在收到端点时可能无法使用完整的Service对象。 服务名称作为key用于协调。
-  它有一个缓存的 EnvoyEndpoint 对象 - 以避免为每个请求和客户端重新分配它。

## Proxy定义

Proxy的定义在文件 `pilot/pkg/model/context.go` 中。虽然 Proxy 的定义不是服务发现的直接组成部分，但是后面发现服务发现的接口中明确出现了Proxy的信息，因此还是将Proxy的定义加进来。

Proxy包含有关代理的特定实例（envoy sidecar，Gateway等）的信息。 当Sidecar连接到Pilot时，代理被初始化，并从协议中的“node”信息以及从注册表中提取的数据填充。

在当前的Istio实现节点中，使用4部分'〜'分隔的ID。格式为 "Type~IPAddress~ID~Domain"。

```go
type Proxy struct {
	// ClusterID 指定代理所在的集群
	// TODO: clarify if this is needed in the new 'network' model, likely needs to be renamed to 'network'
	ClusterID string

	// Type 指定node类型。是 ID 的第一部分
	Type NodeType

	// IPAddresses 是 proxy 用来标识自身和同处一地的服务实例的 IP 地址. 如: "10.60.1.6". 在某些情况下，proxy和服务实例所在的 host 可能有多个IP地址In
	IPAddresses []string

	// ID 是平台特有的sidecar proxy id。对于 k8s，是 pod id 和 namespace。
	ID string

	// Locality 是 envoy proxy 运行的位置
	Locality Locality

	// DNSDomain 为短主机名定义 DNS 域名后缀(如 "default.svc.cluster.local")
	DNSDomain string

	// TrustDomain 定义证书的信任域名
	TrustDomain string

	// ConfigNamespace 定义proxy所在的 namespace，用于网络范围目的
	// NOTE: DO NOT USE THIS FIELD TO CONSTRUCT DNS NAMES
	ConfigNamespace string

	// Metadata 是用于扩展 node 标识符的键值对
	Metadata map[string]string

	// 和proxy关联的 sidecarScope
	SidecarScope *SidecarScope
}
```

Node type的定义有：

```go
// NodeType 决定 proxy 在mesh网络中的责任
type NodeType string

const (
	// SidecarProxy 类型用于在应用容器中的sidecar proxy
	SidecarProxy NodeType = "sidecar"

	// Ingress 类型用于集群的 ingress proxies
	Ingress NodeType = "ingress"

	// Router 类型用于以 L7/L4 路由器方式工作的独立代理
	Router NodeType = "router"
)
```

特意看了一下 Router 类型，支持以下 RouterMode：

```go
// RouterMode 决定 Istio Gateway 的行为 (普通 或者 sni-dnat)
type RouterMode string

const (
	// StandardRouter 是普通的网关模式
	StandardRouter RouterMode = "standard"

	// SniDnatRouter 用于桥接两个网络
	SniDnatRouter RouterMode = "sni-dnat"
)
```

用于桥接两个网络的 SniDnatRouter ，非常有意思，稍后细看。

## Controller定义

Controller定义了一个事件控制器循环。Proxy Agent 向控制器循环注册自身，并接收有关服务拓扑更改或配置工件更改的通知。

控制器保证以下一致性要求：控制器中的注册表视图与当前通知到达时一样新鲜，但可能更新鲜（例如“DELETE”取消“ADD”事件）。例如，创建服务的事件将看到注册表中没有这个服务，如果紧跟这个事件之后有服务删除事件。

Handler 按照它们附加的顺序在单个工作队列上执行。

Handler 接收通知事件和关联的对象。请注意，必须在启动控制器之前附加所有处理程序。


```go
type Controller interface {
	// AppendServiceHandler 通知和服务目录相关的变更
	AppendServiceHandler(f func(*Service, Event)) error

	// AppendInstanceHandler 通知和服务的服务实例相关的变更
	AppendInstanceHandler(f func(*ServiceInstance, Event)) error

	// 运行直到接收到信号
	Run(stop <-chan struct{})
}
```
