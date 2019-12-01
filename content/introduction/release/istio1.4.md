---
date: 2019-04-08T19:00:00+08:00
title: Istio1.4版本
weight: 123
menu:
  main:
    parent: "introduction-release"
description : "Istio1.4版本"
---



## 变更列表

Istio 1.4 的 Change Note 请见：https://istio.io/news/releases/1.4.x/announcing-1.4/change-notes/

下面对这个变更列表做一个详细的解读。

### Traffic management

#### 支持按百分比镜像流量

原文描述：Added support for mirroring a percentage of traffic.

在1.4之前，Istio进行镜像流量时是镜像全部请求，在1.4之后可以通过 `mirror_percent` 参数设置百分比。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
```

为了保持兼容和行为一致，在 `mirror_percent` 参数没有设置的情况下，取默认值 100，即默认所有流量都将被镜像。

#### Envoy在crash时会退出

原文描述：Improved the Envoy sidecar. The Envoy sidecar now exits when it crashes. This change makes it easier to see whether or not the Envoy sidecar is healthy.

改进了Envoy，在crash时会退出（TDB：退出啥？），这样可以方便判断Envoy Sidecar是不是健康的。

### Pilot不再发送重复的配置到Envoy

原文描述：Improved Pilot to skip sending redundant configuration to Envoy when no changes are required.

- Improved headless services to avoid conflicts with different services on the same port.
Disabled default circuit breakers.
- Updated the default regex engine to re2. Please see the Upgrade Notes for details.





## 参考资料

- https://istio.io/news/releases/1.4.x/announcing-1.4/
- https://securityboulevard.com/2019/11/whats-new-in-istio-1-4/amp/
- https://devclass.com/2019/11/15/istio-hits-1-4-and-gets-mixer-less-and-experimental/
- https://www.lizenghai.com/archives/33415.html
- https://haralduebele.blog/2019/11/21/installing-istio-1-4-new-version-new-methods/
- https://jaxenter.de/microservices/isto-1-4-service-mesh-release-89364

