# 架构

## 整体架构

Istio服务网格分为两大块:数据面板和控制面板。

* 数据面板是由一组的作为sidecar部署，调停和控制微服务之间的所有网络通信的智能代理（Envoy）组成的。

* 控制面板 负责管理和配置代理来路由通讯，以及在运行时执行策略。

![](images/arch.svg)

以下分别介绍 istio 中的主要模块 Envoy/Mixer/Pilot/Auth.

> 注: 下面的内容,尤其图片, 很多来自istio官方文档和演讲的PDF.

## Envoy

![](images/envoy.png)

以下介绍内容来自istio官方文档：

> Istio 使用 Envoy 代理的扩展版本，Envoy 是以C++开发的高性能代理，用于调解服务网格中所有服务的所有入站和出站流量。

> Istio利用了Envoy的许多内置功能，例如动态服务发现，负载均衡，TLS termination，HTTP/2&gRPC代理，熔断器，健康检查，基于百分比流量拆分的分段推出，故障注入和丰富的metrics。

> Envoy实现了过滤和路由、服务发现、健康检查，提供了具有弹性的负载均衡。它在安全上支持TLS，在通信方面支持gRPC.

概括说，Envoy 提供的是服务间网络通讯的能力，包括(以下均可支持TLS)：

- HTTP／1.1
- HTTP/2
- gRPC
- TCP

以及网络通讯直接相关的功能：

- 服务发现：从Pilot得到服务发现信息
- 过滤
- 负载均衡
- 健康检查
- 执行路由规则(Rule): 规则来自Polit,包括路由和目的地策略
- 加密和认证: TLS certs来自 istio-Auth

此外, Envoy 也吐出各种数据给Mixer:

- metrics
- logging
- distribution trace: 目前支持 zipkin

总结: Envoy 是 istio 中负责"干活"的模块,如果将整个 istio 体系比喻为一个施工队,那么 Envoy 就是最底层负责搬砖的民工, 所有体力活都由 Envoy 完成. 所有需要控制,决策,管理的功能都是其他模块来负责,然后配置给 Envoy.

### Istio架构回顾

在继续介绍istio其他的模块之前, 我们来回顾一下Istio的架构.前面我们提到, istio 服务网格分为两大块:数据面板和控制面板。

![](images/arch.svg)

我们刚刚介绍的 Envoy, 在istio中扮演的就是数据面板, 而其他我们下面将要陆续介绍的Mixer, Pilot和Auth属于控制面板. 上面我给出了一个类比(有点玩笑性质): istio 中 Envoy (或者说数据面板)扮演的角色是底层干活的民工, 而该让这些民工如何工作, 由包工头控制面板来负责完成.

在istio的架构中,这两个模块的分工非常的清晰,体现在架构上也是经纬分明: Mixer, Pilot和Auth 这三个模块都是Go语言开发,代码托管在github上, 三个仓库分别是 istio/mixer, istio/pilot/auth. 而Envoy来自lyft, 编程语言是c++ 11, 代码托管在github但不是istio下.从团队分工看, google和IBM关注于控制面板中的Mixer, Pilot和Auth, 而Lyft继续专注于Envoy.

Istio的这个架构设计, 将底层service mesh的具体实现,和istio核心的控制面板拆分开. 从而使得istio可以借助成熟的Envoy快速推出产品,未来如果有更好的service mesh方案也方便集成.

### Envoy的竞争者

谈到这里, 我们聊一下目前市面上唯二的Envoy之外的另外一个service mesh成熟产品: 基于scala的 Linkerd: linkerd的功能和定位和 Envoy 非常相似, 而且就在今年上半年成功进入CNCF. 而在 istio 推出之后, linkerd做了一个很有意思的动作: istio推出了和istio的集成, 实际为替换 Envoy 作为istio的数据面板, 和istio的控制面板对接.

回到 istio 的架构图, 将这幅图中的 Envoy 字样替换为 Linkerd 即可. 另外还有不在图中表示的 Linkerd Ingress / Linkerd Egress 用于替代 Envoy 实现 k8s 的Ingress/Egress.

> 注: 一出小三上位原配出局的狗血剧情貌似正在酝酿中. 结局如何我等不妨拭目以待. 还是那句话: 没有挖不倒的墙角, 只有不努力的小三! Linkerd, 加油!

下面开始介绍 istio 中最核心的控制面板.

## Pilot

### 流量管理

istio 最核心的功能是流量管理, 前面我们看到的数据面板, 由Envoy组成的服务网格, 将整个服务间通讯和入口/出口请求都承载于其上.

使用Istio的流量管理模型，本质上将**流量和基础设施扩展解耦**，让运维人员通过Pilot指定他们希望流量遵循什么规则，而不是哪些特定的pod/VM应该接收流量.

对这段话的理解, 可以看下图: 假定我们原有服务B,部署在Pod1/2/3上,现在我们部署一个新版本在Pod4在, 我们希望实现切5%的流量到新版本.

![](images/trafic-control-1.jpg)

如果以基础设施为基础实现上述5%的流量切分,则需要通过某些手段将流量切5%到Pod4这个特定的部署单位, 实施时就必须和serviceB的具体部署还有ServiceA访问ServiceB的特定方式紧密联系在一起. 比如如果两个服务之间是用nginx做反向代理, 则需要增加pod4的ip作为upstream,并调整pod1/2/3/4的权重以实现流量切分.

如果使用istio的流量管理功能, 由于Envoy组成的服务网络完全在istio的控制之下,因此要实现上述的流量拆分非常简单. 假定原版本为1.0, 新版本为2.0, 只要通过 Polit 给Envoy 发送一个规则: 2.0版本5%流量, 剩下的给1.0.

这种情况下, 我们无需关注2.0版本的部署, 也无需改动任何技术设置, 更不需要在业务代码中为此提供任何配置支持和代码修改. 一切由 Pilot 和智能Envoy代理搞定。

我们还可以玩的更炫一点, 比如根据请求的内容来源将流量发送到特定版本:

![](images/trafic-control-2.jpg)

后面我们会介绍如何从请求中提取出 User-Agent 这样的属性来配合规则进行流量控制.

### Pilot的功能概述

我们在前面有强调说, Envoy 在其中扮演的负责搬砖的民工角色, 而指挥Envoy工作的民工头就是 Pilot 模块.

官方文档中对 Pilot 的功能描述:

> Pilot 负责收集和验证配置并将其传播到各种Istio组件。它从Mixer和Envoy中抽取环境特定的实现细节，为他们提供独立于底层平台的用户服务的抽象表示。此外，流量管理规则（即通用4层规则和7层HTTP/gRPC路由规则）可以在运行时通过Pilot进行编程。

> 每个Envoy实例根据其从Pilot获得的信息以及其负载均衡池中的其他实例的定期健康检查来维护 负载均衡信息，从而允许其在目标实例之间智能分配流量，同时遵循其指定的路由规则。

> Pilot负责在Istio服务网格中部署的Envoy实例的生命周期。

### Pilot的架构

下图是 Pilot 的架构图:

![](images/PilotAdapters.svg)

1. Envoy API 负责和Envoy的通讯, 主要是发送服务发现信息和流量控制规则给Envoy
2. Envoy 提供服务发现，负载均衡池和路由表的动态更新的API。这些API将 istio 和 Envoy 的实现解耦。(另外,也使得 Linkerd 之类的其他服务网络实现得以平滑接管Envoy)
3. Polit 定了一个抽象模型, 以从特定平台细节中解耦, 为跨平台提供基础.
4. Platform Adapter 则是这个抽象模型的现实实现版本, 用于对接外部的不同平台
5. 最后是 Rules API, 提供接口给外部调用以管理 Pilot, 包括命令行工具 istioctl 以及未来可能出现的第三方管理界面

### 服务规范和实现

Pilot 架构中, 最重要的是 Abstract Model 和 Platform Adapter, 我们详细介绍.

- Abstract Model: 是对服务网格中"服务"的规范表示, 即定义在istio中什么是服务, 这个规范独立于底层平台.
- Platform Adapter: 这里有各种平台的实现, 目前主要是Kubernetes, 另外最新的0.2版本的代码中出现了 Consul 和 Eureka.

我们看一下 Pilot 0.2 的代码, pilot/platform 目录下:

![](images/platform-list.jpg)

瞄一眼 platform.go:

```go
// ServiceRegistry 定义支持服务注册的底层平台
type ServiceRegistry string

const (
	// KubernetesRegistry environment flag
	KubernetesRegistry ServiceRegistry = "Kubernetes"
	// ConsulRegistry environment flag
	ConsulRegistry ServiceRegistry = "Consul"
	// EurekaRegistry environment flag
	EurekaRegistry ServiceRegistry = "Eureka"
)
```

服务规范的定义在 modle/service.go 中:

```go
type Service struct {
	Hostname string `json:"hostname"`
	Address string `json:"address,omitempty"`
	Ports PortList `json:"ports,omitempty"`
	ExternalName string `json:"external"`
	ServiceAccounts []string `json:"serviceaccounts,omitempty"`
}
```

由于时间有限, 代码部分我们不深入, 只是通过上面的两段代码来展示 pilot 中对服务的规范定义和目前的几个实现.

暂时而言(当前版本是0.1.6, 0.2版本尚未正式发布), 目前 istio 只支持k8s一种服务发现机制.

备注: Consul 的实现据说主要是为了支持后面将要支持的Cloud Foundry, Eureka 没有找到资料. Etcd3 的支持还在 issue 列表中, 看issue记录争执中.

### pilot功能

基于上述的架构设计, pilot 提供以下重要功能:

- 请求路由
- 服务发现和负载均衡
- 故障处理
- 故障注入
- 规则配置

由于时间限制, 今天不逐个展开详细介绍每个功能的详情. 大家通过名字就大概可以知道是什么, 如果希望了解详情可以关注之后的分享. 或者查阅官方文档的介绍.

## Mixer

Mixer 翻译成中文是 混音器, 下面是它的图标:

![](images/mixer-logo.png)

功能概括: Mixer 负责在服务网格上执行访问控制和使用策略，并收集Envoy代理和其他服务的遥测数据。

### Mixer的设计背景

我们的系统通常会基于大量的基础设施而构建, 这些基础设施的后端服务为业务服务提供各种支持功能。包括访问控制系统，遥测捕获系统，配额执行系统，计费系统等。在传统设计中, 服务直接与这些后端系统集成，容易产生硬耦合.

在istio中,为了避免应用程序的微服务和基础设施的后端服务之间的耦合, 提供了 Mixer 作为两者的通用中介层:

![](images/mixer-traffic.svg)

Mixer 设计将策略决策从应用层移出并用配置替代，并在运维人员控制下。应用程序代码不再将应用程序代码与特定后端集成在一起，而是与Mixer进行相当简单的集成，然后 Mixer 负责与后端系统连接。

特别提醒: Mixer **不是**为了在基础设施后端之上创建一个抽象层或者可移植性层。也不是试图定义一个通用的logging API，通用的metric API，通用的计费API等等。

Mixer的设计目标是减少业务系统的复杂性，**将策略逻辑从业务的微服务的代码转移到Mixer中**, 并且改为让运维人员控制。

### Mixer的功能

Mixer 提供三个核心功能：

- 前提条件检查。允许服务在响应来自服务消费者的传入请求之前验证一些前提条件。前提条件包括认证，黑白名单，ACL检查等等。

- 配额管理。使服务能够在多个维度上分配和释放配额。典型例子如限速(Rating Limit)。

- 遥测报告。使服务能够上报日志和监控。

在Istio内，Envoy 重度依赖 Mixer。

### Mixer的适配器

Mixer 是高度模块化和可扩展的组件。其中一个关键功能是抽象出不同策略和遥测后端系统的细节，允许 Envoy 和基于 Istio 的服务与这些后端无关，从而保持他们的可移植。

Mixer在处理不同基础设施后端的灵活性是通过使用通用插件模型实现的。单个的插件被称为适配器，它们允许 Mixer 与不同的基础设施后端连接，这些后台可提供核心功能，例如日志，监控，配额，ACL检查等。适配器使Mixer能够暴露一致的API，与使用的后端无关。在运行时通过配置确定确切的适配器套件，并且可以轻松指向新的或定制的基础设施后端。

![](images/adapters.svg)

这个图是官网给的, 不是很好理解, 我从 github 的代码中抓个图给大家展示一下目前已有的mixer adapter:

![](images/mixer-adapter-list.jpg)

### Mixer的工作方式

Istio使用 属性 来控制在服务网格中运行的服务的运行时行为。属性是描述入口和出口流量的有名称和类型的元数据片段，以及此流量发生的环境。Istio属性携带特定信息片段，例如：

```bash
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
target.service: example
```

请求处理过程中, 属性由 Envoy 收集并发送给 Mixer, Mixer 中根据运维人员设置的配置来处理属性. 基于这些属性，Mixer会产生对各种基础设施后端的调用。

![](images/machine.svg)

Mixer设计有一套强大(也很复杂, 堪称istio中最复杂的一个部分)的配置模型来配置适配器的工作方式, 设计有适配器, 切面, 属性表达式, 选择器, 描述符,manifests 等一堆概念.

由于时间所限,今天不展开这块内容, 这里给出两个简单的例子让大家对mixer的配置有所认识:

1. 这是一个ip地址检查的adapter.实现类似黑名单或者白名单的功能:

    ```bash
    adapters:
      - name: myListChecker     # 这个配置块的用户定义的名称
        kind: lists             # 这个适配器可以使用的切面类型
        impl: ipListChecker     # 要使用的特定适配器组件的名称
        params:
          publisherUrl: https://mylistserver:912
          refreshInterval: 60s
    ```

2. metrics的适配器,将数据报告给Prometheus系统

    ```bash
    adapters:
      - name: myMetricsCollector
        kind: metrics
        impl: prometheus
    ```

3. 定义切面, 使用前面定义的 myListChecker 这个adapter 对属性 source.ip 进行黑名单检查

    ```bash
    aspects:
    - kind: lists               # 切面的类型
      adapter: myListChecker    # 实现这个切面的适配器
      params:
        blacklist: true
        checkExpression: source.ip
    ```

## Istio-Auth

Istio-Auth 提供强大的服务到服务和最终用户认证，使用交互TLS，内置身份和凭据管理。它可用于升级服务网格中的未加密流量，并为运维人员提供基于服务身份而不是网络控制实施策略的能力。

Istio的未来版本将增加细粒度的访问控制和审计，以使用各种访问控制机制（包括基于属性和角色的访问控制以及授权钩子）来控制和监视访问您的服务，API或资源的人员。

### auth的架构

下图展示 Istio Auth 架构，其中包括三个组件：身份，密钥管理和通信安全。

![](images/auth.svg)

在这个例子中, 服务A以服务帐户“foo”运行, 服务B以服务帐户“bar”运行, 他们之间的通讯原来是没有加密的. 但是 istio 在不修改代码的情况, 依托 Envoy 形成的服务网格, 直接在客户端 envoy 和服务器端 envoy 之间进行通讯加密。

目前在 Kubernetes 上运行的 istio，使用 Kubernetes service account/服务帐户 来识别运行该服务的人员.

### 未来将推出的功能

auth在目前的istio版本(0.1.6和即将发布的0.2)中,功能还不是很全, 未来则规划有非常多的特性:

- 细粒度授权和审核
- 安全Istio组件（Mixer, Pilot等）
- 集群间服务到服务认证
- 使用 JWT/OAuth2/OpenID_Connect 终端到服务的认证
- 支持GCP服务帐户和AWS服务帐户
- 非http流量（MySql，Redis等）支持
- Unix域套接字，用于服务和Envoy之间的本地通信
- 中间代理支持
- 可插拔密钥管理组件

