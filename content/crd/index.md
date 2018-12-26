---
date: 2018-12-26T18:00:00+08:00
title: Istio CRD
weight: 2000
description : "介绍istio的CRD"
---

Istio 中的 CRD 有50多个。

### Network

| 名称              | 全称                                  | 用途                              | 分类       | 归属  |
| ----------------- | ------------------------------------- | --------------------------------- | ---------- | ----- |
| VirtualService    | virtualservices.networking.istio.io   | 用于路由，定义virtual service     | networking | pilot |
| DestinationRule   | destinationrules.networking.istio.io  | 用于路由，定义destination rule    | networking | pilot |
| Gateway           | serviceentries.networking.istio.io    | 用于路由，定义service entry       | networking | pilot |
| ServiceEntry      | gateways.networking.istio.io          | 用于路由，定义gateway             | networking | pilot |
| EnvoyFilter       | envoyfilters.networking.istio.io      | 使用filter为特定envoy添加特定配置 | networking | pilot |
| ServiceDependency | servicedependency.networking.istio.io |                                   | networking | pilot |

相关资料：

- 相关代码在 `istio/api` 仓库下的 `network/v1alpha3` 
- [配置的参考文档](https://preliminary.istio.io/docs/reference/config/istio.networking.v1alpha3/)

### Authentication

| 名称        | 全称                                 | 用途                        | 分类           | 归属    |
| ----------- | ------------------------------------ | --------------------------- | -------------- | ------- |
| Policy      | policies.authentication.istio.io     | 用于auth，作用域为namespace | authentication | citadel |
| Mesh Policy | meshpolicies.authentication.istio.io | 用于auth，作用域为global    | authentication | citadel |

### Config

都是用于mixer。

#### API Management

| 名称 | 全称                                | 用途 | 分类            | 归属  |
| ---- | ----------------------------------- | ---- | --------------- | ----- |
|      | httpapispecbindings.config.istio.io |      | api managerment | mixer |
|      | httpapispecs.config.istio.io        |      | api managerment | mixer |
|      | quotaspecbindings.config.istio.io   |      | api managerment | mixer |
|      | quotaspecs.config.istio.io          |      | api managerment | mixer |

#### Mixer Core

| 名称 | 全称                               | 用途 | 分类       | 归属  |
| ---- | ---------------------------------- | ---- | ---------- | ----- |
|      | rules.config.istio.io              |      | mixer core | mixer |
|      | attributemanifests.config.istio.io |      | mixer core | mixer |

#### RBAC

| 名称 | 全称                              | 用途                                | 分类 | 归属  |
| ---- | --------------------------------- | ----------------------------------- | ---- | ----- |
|      | rbacconfigs.rbac.istio.io         | 用于authz，定义istio的rbac策略      | rbac | mixer |
|      | serviceroles.rbac.istio.io        | 用于authz，定义service role         | rbac | mixer |
|      | servicerolebindings.rbac.istio.io | 用于authz，定义service role binding | rbac | mixer |

#### Mixer Adapter

用于处理从envoy收集的数据

| 名称 | 全称                            | 用途                       | 分类          | 归属  |
| ---- | ------------------------------- | -------------------------- | ------------- | ----- |
|      | bypasses.config.istio.io        | 定义bypass adapter         | mixer adapter | mixer |
|      | circonuses.config.istio.io      | 定义circonus adapter       | mixer adapter | mixer |
|      | cloudwatch.config.istio.io      | 定义cloud watch adapter    | mixer adapter | mixer |
|      | deniers.config.istio.io         | 定义dinier adapter         | mixer adapter | mixer |
|      | dogstatsd.config.istio.io       | 定义dogstatsd adapter      | mixer adapter | mixer |
|      | fluentds.config.istio.io        | 定义fluentd adapter        | mixer adapter | mixer |
|      | kubernetesenvs.config.istio.io  | 定义kubernetesenv adapter  | mixer adapter | mixer |
|      | listcheckers.config.istio.io    | 定义list adapter           | mixer adapter | mixer |
|      | memquotas.config.istio.io       | 定义memquota adapter       | mixer adapter | mixer |
|      | noops.config.istio.io           | 定义noops adapter          | mixer adapter | mixer |
|      | opas.config.istio.io            | 定义opa adapter            | mixer adapter | mixer |
|      | prometheuses.config.istio.io    | 定义prometheus adapter     | mixer adapter | mixer |
|      | rbacs.config.istio.io           | 定义rbac adapter           | mixer adapter | mixer |
|      | redisquotas.config.istio.io     | 定义redisquota adapter     | mixer adapter | mixer |
|      | servicecontrols.config.istio.io | 定义servicecontrol adapter | mixer adapter | mixer |
|      | signalfxs.config.istio.io       | 定义signalfx adapter       | mixer adapter | mixer |
|      | solarwinds.config.istio.io      | 定义solarwinds adapter     | mixer adapter | mixer |
|      | stackdrivers.config.istio.io    | 定义stackdriver adapter    | mixer adapter | mixer |
|      | statsds.config.istio.io         | 定义statsd adapter         | mixer adapter | mixer |
|      | stdios.config.istio.io          | 定义stdio adapter          | mixer adapter | mixer |

#### Mixer Template

用于定义从envoy收集的数据

| 名称 | 全称                           | 用途                       | 分类           | 归属  |
| ---- | ------------------------------ | -------------------------- | -------------- | ----- |
|      | apikeys.config.istio.io        | 定义apikey template        | mixer template | mixer |
|      | authorizations.config.istio.io | 定义authorization template | mixer template | mixer |
|      | checknothings.config.istio.io  | 定义checknothing template  | mixer template | mixer |
|      | edges.config.istio.io          | 定义edge template          | mixer template | mixer |
|      | listentries.config.istio.io    | 定义listentry template     | mixer template | mixer |
|      | logentries.config.istio.io     | 定义logentry template      | mixer template | mixer |
|      | metrics.config.istio.io        | 定义metric template        | mixer template | mixer |
|      | quotas.config.istio.io         | 定义quota template         | mixer template | mixer |
|      | reportnothings.config.istio.io | 定义reportnothing template | mixer template | mixer |
|      | tracespans.config.istio.io     | 定义tracespan template     | mixer template | mixer |

#### Others

| 名称 | 全称                      | 用途 | 分类   | 归属  |
| ---- | ------------------------- | ---- | ------ | ----- |
|      | adapters.config.istio.io  |      | others | mixer |
|      | instances.config.istio.io |      | others | mixer |
|      | templates.config.istio.io |      | others | mixer |
|      | handlers.config.istio.io  |      | others | mixer |



### 参考资料

- 沈旭光 同学的文档  [Istio CRD Cheatsheet](https://docs.google.com/spreadsheets/d/14eXerRWNsCJDUrKWjoQIwwxbB8TnvrbgQQvC0RmWFD4/edit#gid=0)
- 宋净超 同学的 [istio handbook](https://jimmysong.io/istio-handbook/setup/istio-installation.html)