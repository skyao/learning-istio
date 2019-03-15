---
date: 2019-03-15T15:00:00+08:00
title: Sidecar
weight: 417
menu:
  main:
    parent: "crd-network"
description : "Sidecar CRD"
---

## 介绍

官方文档：

- https://istio.io/docs/reference/config/istio.networking.v1alpha3/#Sidecar

Sidecar描述了sidecar代理的配置，sidecar代理调解与其连接的工作负载的 inbound 和 outbound 通信。 默认情况下，Istio将为网格中的所有Sidecar代理服务，使其具有到达网格中每个工作负载所需的必要配置，并在与工作负载关联的所有端口上接收流量。 Sidecar资源提供了一种的方法，在向工作负载转发流量或从工作负载转发流量时，微调端口集合和代理将接收的协议。 此外，可以限制代理在从工作负载转发 outbound 流量时可以达到的服务集合。

网格中的服务和配置被组织成一个或多个名称空间（例如，Kubernetes名称空间或CF org/space）。 命名空间中的Sidecar资源将应用于同一命名空间中的一个或多个工作负载，由workloadSelector选择。 如果没有workloadSelector，它将应用于同一名称空间中的所有工作负载。 在确定要应用于工作负载的Sidecar资源时，将优先使用通过workloadSelector而选择到此工作负载的的资源，而不是没有任何workloadSelector的资源。

> 注意：每个命名空间只能有一个没有任何工作负载选择器的Sidecar资源。 如果给定命名空间中存在多个无选择器的Sidecar资源，则系统的行为是不确定的。 如果具有工作负载选择器的两个或多个Sidecar资源选择相同的工作负载，则系统的行为是不确定的。

下面的示例在prod-us1命名空间中声明了Sidecar资源，该资源配置命名空间中的sidecars以允许出口流量到prod-us1，prod-apis和istio-system命名空间中的公共服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  egress:
  - hosts:
    - "prod-us1/*"
    - "prod-apis/*"
    - "istio-system/*"
```

下面的示例在prod-us1命名空间中声明了一个Sidecar资源，该资源接受端口9080上的入站HTTP流量，并将其转发到在Unix域套接字上侦听的连接工作负载。 在出口方向上，除了istio-system名称空间之外，sidecar仅代理为prod-us1名称空间中的服务绑定到端口9080的HTTP流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  ingress:
  - port:
      number: 9080
      protocol: HTTP
      name: somename
    defaultEndpoint: unix:///var/run/someuds.sock
  egress:
  - port:
      number: 9080
      protocol: HTTP
      name: egresshttp
    hosts:
    - "prod-us1/*"
  - hosts:
    - "istio-system/*"
```

如果部署工作负载时没有基于 iptable 的流量捕获，则Sidecar资源是配置和工作负载连接的代理端口的唯一方法。 以下示例在 `prod-us1` namespace 中为所有带有标签“app:productpage”并属于 `productpage.prod-us1` 服务的pod声明了Sidecar资源。 假设这些pod是在没有IPtable规则（即Istio init container）的情况下部署的，并且代理元数据ISTIOMETAINTERCEPTION_MODE设置为NONE，则下面的规范允许此类pod在端口9080上接收HTTP流量并将其转发到在 127.0.0.1:8080 上监听的。 它还允许应用程序与127.0.0.1:3306上的MySQL数据库进行通信，然后在mysql.foo.com:3306上代理到外部托管的MySQL服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: no-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - port:
      number: 9080 # binds to 0.0.0.0:9080
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    captureMode: NONE # not needed if metadata is set for entire proxy
  egress:
  - port:
      number: 3306
      protocol: MYSQL
      name: egressmysql
    captureMode: NONE # not needed if metadata is set for entire proxy
    bind: 127.0.0.1
    hosts:
    - "*/mysql.foo.com"
```

以及用于路由到 mysql.foo.com:3306 的相关service entry：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mysql
  namespace: ns1
spec:
  hosts:
  - mysql.foo.com
  ports:
  - number: 3306
    name: mysql
    protocol: MYSQL
  location: MESH_EXTERNAL
  resolution: DNS
```

还可以在单个代理中混合和匹配流量捕获模式。 例如，考虑内部服务位于192.168.0.0/16子网上的安装。 此时，在VM上设置IP table 以捕获 192.168.0.0/16 子网上的所有出站流量。 假设VM在 172.16.0.0/16 子网上具有用于 inbound 流量的额外网络接口。 以下Sidecar配置允许VM在172.16.1.32:80（VM的IP）上暴露监听器，以获取到达172.16.0.0/16子网的流量。 请注意，在此方案中，VM中代理上的ISTIOMETAINTERCEPTION_MODE元数据应包含“REDIRECT”或“TPROXY”作为其值，这意味着基于IP tables的流量捕获处于活动状态。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: partial-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - bind: 172.16.1.32
    port:
      number: 80 # binds to 172.16.1.32:80
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    captureMode: NONE
  egress:
    # use the system detected defaults
    # sets up configuration to handle outbound traffic to services
    # in 192.168.0.0/16 subnet, based on information provided by the
    # service registry
  - captureMode: IPTABLES
    hosts:
    - "*/*"
```


## CRD定义

### Sidecar

| Field              | Type                     | Description                                                  |
| ------------------ | ------------------------ | ------------------------------------------------------------ |
| `workloadSelector` | `WorkloadSelector`       | 用于选择应该应用此sidecar配置的特定pod/VM集的条件。 如果省略，则sidecar配置将应用于同一config命名空间中的所有工作负载。 |
| `ingress`          | `IstioIngressListener[]` | Ingress指定Sidecar的配置，用于处理连接工作负载的 inbound 流量。 如果省略，Istio将根据从编排平台获得的工作负载信息（例如，暴露的端口，服务等）自动配置Sidecar。 如果指定，则当且仅当工作负载与服务关联时，才会配置inbound端口。 |
| `egress`           | `IstioEgressListener[]`  | Egress指定 Sidecar 的配置，用于处理从相关工作负载到网格中其他服务的 outbound 流量。 如果省略，Istio将自动配置Sidecar，以便能够到达此命名空间可见的网格中的每个服务。 |

### WorkloadSelector

WorkloadSelector指定用于确定Gateway或Sidecar资源是否可以应用于代理的条件。 匹配条件包括与代理相关联的元数据，工作负载信息（例如附加到pod / VM的标签）或代理在初始握手期间提供给Istio的任何其他信息。 如果指定了多个条件，则所有条件都需要匹配才能选择工作负载。 目前，仅支持基于标签的选择机制。

| Field    | Type                  | Description                                                  |
| -------- | --------------------- | ------------------------------------------------------------ |
| `labels` | `map<string, string>` | 必需：一个或多个标签，指示应该应用此Sidecar配置的特定pod/VM集合。 标签搜索的范围仅限于资源所在的配置命名空间。 |



### IstioIngressListener

IstioIngressListener指定连接到工作负载的sidecar代理上的 inbound 流量监听器的属性。

| Field             | Type          | Description                                                  |
| ----------------- | ------------- | ------------------------------------------------------------ |
| `port`            | `Port`        | 必须。与监听器关联的端口。 如果使用Unix domain socket，请使用0作为端口号，并使用有效的协议。 |
| `bind`            | `string`      | 应该绑定监听器的 ip 或Unix domain socket。 格式：`x.x.x.x` 或 `unix:///path/to/uds` 或 `unix://@foobar`（Linux抽象命名空间）。 如果省略，Istio将根据导入的服务以及应用此配置的工作负载自动配置默认值。 |
| `captureMode`     | `CaptureMode` | 当绑定地址是IP时，captureMode选项指示如何劫持（或不劫持）到监听器的流量。 对于Unix domain socket，captureMode必须为DEFAULT或NONE。 |
| `defaultEndpoint` | `string`      | 必需：应将流量转发到的环回IP端点或Unix domain socket。 此配置可用于将到达Sidecar上绑定端口的流量重定向到应用程序工作负载正在监听连接的端口或Unix domain socket。 格式应为 `127.0.0.1:PORT` 或 `unix:///path/to/socket` |

### IstioEgressListener

IstioEgressListener指定附加到工作负载的sidecar代理上的 outbound 流量监听器的属性。

| Field         | Type          | Description                                                  |
| ------------- | ------------- | ------------------------------------------------------------ |
| `port`        | `Port`        | 与监听器关联的端口。 如果使用Unix域套接字，请使用0作为端口号，并使用有效的协议。 端口（如果已指定）将用作与导入的host关联的默认目标端口。 如果省略端口，Istio将根据导入的host推断监听器端口。 请注意，当指定多个 egress 监听器时，其中一个或多个监听器具有特定端口而其他监听器没有端口，则监听器端口上暴露的host将基于具有最特定端口的监听器。（the hosts exposed on a listener port will be based on the listener with the most specific port，不理解，后面继续看） |
| `bind`        | `string`      | 应该绑定监听器的 ip 或Unix domain socket。 格式：`x.x.x.x` 或 `unix:///path/to/uds` 或 `unix://@foobar`（Linux抽象命名空间）。 如果省略，Istio将根据导入的服务以及应用此配置的工作负载和captureMode自动配置默认值。 |
| `captureMode` | `CaptureMode` | 当绑定地址是IP时，captureMode选项指示如何劫持（或不劫持）到监听器的流量。 对于Unix Domain Socket绑定，captureMode必须是 DEFAULT 或 NONE。 |
| `hosts`       | `string[]`    | **必需**：以 `namespace/dnsName` 格式被监听器暴露的的一个或多个服务主机。在指定namespace内与dnsName匹配的服务将被暴露（也就是可以访问）。相应的服务可以是服务注册表中的服务（例如，Kubernetes或cloud foundry服务）或使用ServiceEntry或 VirtualServicec 配置指定的服务。还可以使用同一名称空间中的任何关联的DestinationRule。<br/><br/>应使用FQDN格式指定dnsName，在最左侧的组件中可以包含通配符（例如，`prod / * .example.com`）。将dnsName设置为 `*` 可以从指定的命名空间中选择所有服务（例如，`prod/*.example.com`）。命名空间也可以设置为 `*` 以从任何可用的命名空间中选择特定服务（例如，`"*/ foo.example.com"`）。<br/><br/>注意：只有已导出到 sidecar 命名空间的服务和配置工件（例如，exportTo值为`*`）可以应用。私有配置（例如，exportTo 设置为 `.`）将不可用。有关详细信息，请参阅“VirtualService”，“DestinationRule”和“ServiceEntry”配置中的 `exportTo` 设置。 |


### CaptureMode

CaptureMode描述了如何捕获到监听器的流量。 仅在监听器绑定到IP时才适用。

| Name       | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `DEFAULT`  | 环境定义的默认劫持模式                                       |
| `IPTABLES` | 使用IPtables重定向劫持流量                                   |
| `NONE`     | 没有流量劫持。在egress listener中使用时，应用程序应该与监听器端口/unix domain socket 明确通信。 在ingress监听器中使用时，需要注意确保监听器端口未被主机上的其他进程使用。 |