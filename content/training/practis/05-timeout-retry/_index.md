---
title: "超时重试"
linkTitle: "超时重试"
weight: 50
date: 2024-06-18
description: >
  超时重试: 提升系统的健壮性和可用性
---

## 超时

参考官方文档：

- https://istio.io/latest/docs/tasks/traffic-management/request-timeouts/


使用 reviews v2 版本：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
EOF
```

通过故障注入功能，为 ratings 服务增加2秒的延迟：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 2s
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

打开浏览器访问页面感受一下延迟。

然后通过 reviews 的 VirtualService，设置 timeout 为 0.5s （小于ratings的应答延迟）：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 0.5s
EOF
```

这样，reviews 因为要访问延迟为2秒的 ratings 服务，必然会无法满足 0.5 秒延迟的要求，从而报错。

## 重试

参考官方文档：

- https://istio.io/latest/docs/concepts/traffic-management/#retries

首先为 ratings 服务注入故障，让其返回500，但是概率设置为 50%：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 50
    route:
    - destination:
        host: ratings
        subset: v1
EOF
```

然后多次刷新页面，感受50%概率的报错。

为 ratings 服务设置重试，重试次数依次为2/3/4，感受重试之后报错概率的降低：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 50
    route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 1
EOF
```

> 备注：这个验证失败了，后来发现时 istio 不能同时使用故障注入和retry
> 
> Currently, the fault injection configuration can not be combined with retry or timeout configuration on the same virtual service, see Traffic Management Problems.

这个 issue 详细讨论了这个问题：

https://github.com/istio/istio/issues/13705

解决方案在这里，改用 envoy filter进行故障注入：

https://github.com/istio/istio.io/pull/10781/files

因此需要删除在 VirtualService 中进行故障注入的代码，只保留重试的设置（注意要加retryOn: 5xx）：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 1
      retryOn: 5xx
EOF
```

然后利用 EnvoyFilter 进行故障注入：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ratings-filter
spec:
  workloadSelector:
    labels:
      app: ratings
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND # will match outbound listeners in all sidecars
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.fault
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault"
          abort:
            http_status: 500
            percentage:
              numerator: 50
              denominator: HUNDRED
EOF
```

### 演示精确统计错误概率

简单统计一下出错的概率，为此我们需要找到方法。

首先这个命令组合可以看到是否成功：

```bash
curl -sS http://192.168.6.224:30678/productpage | grep unavailable
```

如果出现失败，则输出为：

```bash
        <p><i>Ratings service is currently unavailable</i></p>
        <p><i>Ratings service is currently unavailable</i></p>
```

这个命令后面再加一个 wc -l

```bash
curl -sS http://192.168.6.224:30678/productpage | grep unavailable | wc -l 
```

则成功输出为0，失败输出为2 （两个 unavailable 字眼）。

重复 100 次，则可以看到 100个 0 或者2 的输出：


```bash
for i in $(seq 1 100); do curl -sS http://192.168.6.224:30678/productpage | grep unavailable | wc -l ; done
```

继续 grep 2 后 wc，这样就能得到错误发生的次数：

```bash
for i in $(seq 1 100); do curl -sS http://192.168.6.224:30678/productpage | grep unavailable | wc -l ; done | grep 2 | wc -l
```

这个数字，如 52，和总次数 100 相比，就能得到错误发生的概率。

每次请求有 50% 的概率失败，则：

- 不重试失败概率为 50%
- 重试一次失败概率为 25%
- 重试两次失败概率为 12.5%
- 重试三次失败概率为 6%

