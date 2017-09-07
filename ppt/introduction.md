# 介绍

## istio是什么 what?

官方解释:

Istio：一个连接，管理和保护微服务的开放平台。

Istio为希腊语，意思是"sail"/“启航”。

![](../overview/images/istio.png)

Istio提供了一种轻松的方式来创建一个已部署服务的具有负载平衡，服务到服务认证，监控等等的网络，而不需要任何服务代码的改动。

### 主要特性

HTTP、gRPC和TCP网络流量的自动负载均衡；
提供了丰富的路由规则，实现细粒度的网络流量行为控制；
流量加密、服务间认证，以及强身份声明；
全范围（Fleet-wide）策略执行；
深度遥测和报告。

关键Istio功能:

HTTP / 1.1，HTTP / 2，gRPC和TCP流量的自动区域感知负载平衡和故障切换。
通过丰富的路由规则，容错和故障注入，对流行为的细粒度控制。
支持访问控制，速率限制和配额的可插拔策略层和配置API。
集群内所有流量的自动量度，日志和跟踪，包括集群入口和出口。
安全的服务到服务身份验证，在集群中的服务之间具有强大的身份标识。

## 为什么要使用Istio？ why

### 需要解决的问题

微服务简化了开发，它将创建复杂系统的任务切分为数十乃至上百个小服务，这些小服务易于被小型的软件工程师团队所理解和修改。

但是微服务并未真正地消除复杂性，而是将复杂性迁移到对大量服务的连接、管理和监控上。

其中涉及对上百个服务的管理、处理部署问题、版本控制、安全、故障转移、策略执行、遥测和监控等，实现它们并非易事。Istio力图去解决这些问题。

Istio解决了开发人员和运维在单体应用程序向分布式微服务架构的转型中面临的许多挑战。

术语服务网格通常用于描述构成这些应用程序的微服务网络以及它们之间的交互。随着服务网格的规模和复杂性的增长，它变得更难以理解和管理。

它的需求包括服务发现，负载均衡，故障恢复，指标和监控，以及通常更复杂的运维需求，例如A/B测试，金丝雀发布，限流，访问控制和端到端认证。

### 如何解决?

按Google的提法，Istio是“架构的一层，处于服务和网络间”，它“通常连同服务部署一起，统称为服务啮合层（Service Mesh）”。在Istio网站上，详细解释了“服务啮合层”这一概念：

	如果我们可以在架构中的服务和网络间透明地注入一层，那么该层将赋予操作人员对所需功能的控制，同时将开发人员从编码实现分布式系统问题中解放出来。通常将该统一的架构层与服务部署一起，统称为一个“服务啮合层”。由于微服务有助于分离各个特性团队（Feature Team），因此服务啮合层有助于将操作人员从应用特性开发和发布过程中分离出来。通过系统地注入代理到微服务间的网络路径中，Istio将迥异的微服务转变成一个集成的服务啮合层。

#### 服务网格

Istio提供了一个完整的解决方案，通过为整个服务网格提供行为洞察和操作控制来满足微服务应用程序的多样化需求。它在服务网络中统一提供了许多关键功能：

通讯管理。控制服务之间的通讯和API调用的流量，使得调用更可靠，并使网络在恶劣情况下更加健壮。

可观察性。了解服务之间的依赖关系，以及它们之间通讯的性质和流量，从而提供快速识别问题的能力。

策略执行。将组织策略应用于服务之间的互动，确保访问策略得到实施，资源在消费者之间分配。通过配置网格而不是修改应用程序代码来进行策略更改。

服务身份和安全。在网格中的服务提供具有可验证身份，并提供保护在不同程度的可信度网络上流转的服务通讯的能力。

除了这些行为，Istio还设计了可扩展性以满足不同的部署需求：



集成和定制。可以扩展和定制策略执行组件，以便与现有的ACL，日志，监控，配额，审核等解决方案集成。

这些功能大大减少了应用程序代码，底层平台和策略之间的耦合。这种减少的耦合不仅使服务更容易实现，而且还使运维人员更容易地在环境之间移动应用程序部署或新的策略方案。因此，结果就是应用程序从本质上变得更容易移动。


## istio who

2017年5月,Google、IBM和Lyft开源了Istio

## 支持

### 运行环境支持

Istio旨在在各种环境中运行，包括span Cloud, on-premise，Kubernetes，Mesos等各种环境。我们最初专注于Kubernetes，但正在努力很快支持其他环境。

Istio支持团队计划将其与Google Cloud Endpoints和Apigee集成。

此外，Red Hat、Pivotal、Weaveworks、Tigera和Datawire也有兴趣将自身的产品与Istio集成。

Istio 1.0版计划于今年下半年发布。

我们计划每3个月重新发布一次新版本

- 0.1 版本2017年5月发布,只支持Kubernetes
- 0.2 即将发布, 当前是0.2.1 pre-release, 也只支持Kubernetes
- 0.3 roadmap说要支持k8s之外的平台, "Support for Istio meshes without Kubernetes."
- 1.0 版本预计今年年底发布 "we invite the community to join us in shaping the project as we work toward a 1.0 release later this year."

### 团队支持

istio的开发团队主要来自 google, IBM 和 Lyft.

基于我们为内部和企业客户构建和运营大规模微服务的常见经验，Google，IBM和Lyft联手创建Istio，希望为微服务开发和维护提供可靠的基础。

Google和IBM在自己的应用程序中与这些大型微服务器以及其敏感/监管环境中的企业客户有丰富的经验，而Lyft开发了Envoy来解决其内部可操作性挑战。

举例,下图是istio Working Group的成员列表,

![](images/working_group.jpg)

总共18人,10个google和8个IBM.

而Lyft的贡献主要在Envoy.

#### google

云服务平台产品经理 Varun Talwar

Istio封装了Google一直在使用的许多最佳做法，用于在生产中运行大规模服务多年。我们很高兴为社区做出贡献，作为与Kubernetes合作的开放解决方案;

Google Cloud is committed to open-source, whether it’s bringing new technologies in the open like Kubernetes or gRPC; contributing to projects like Envoy; or supporting open-source tools on Google Cloud Platform.

Google hardened Envoy on several aspects related to security, performance, and scalability.

- grpc 标准化RPC库
- REST/http

Istio is the latest instance of Google's continuing contribution to open-source as part of a collaborative community effort.

#### IBM

Jason McGee,IBM研究员，IBM云平台副总裁兼CTO:

https://developer.ibm.com/dwblog/2017/istio/

“IBM很高兴与Google合作推出Istio项目，并为云开发人员提供将不同的微服务转变为综合服务网格所需的工具。

Istio目前在Kubernetes平台上运行，如IBM Bluemix Container Service。



IBM的贡献:

 In addition to developing the Istio control plane, IBM also contributed several features to Envoy such as traffic splitting across service versions, distributed request tracing with Zipkin and fault injection.

#### Lyft

Lyft 的经过实战测试的 Envoy 代理构建

Lyft开源的Envoy在成功使用它一年以上，可以管理超过100个服务跨越10,000个虚拟机，处理2M请求/秒。

### 社区支持

目前已经有社区在提供对istio的支持或者集成:

- Red Hat: Openshift and OpenShift Application Runtimes
- Pivotal: Cloud Foundry
- Weaveworks: Weave Cloud and Weave Net 2.0
- Tigera: Project Calico Network Policy Engine
- Datawire: Ambassador project

其他外围支持:

- eureka
- consul


