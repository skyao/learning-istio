---
title: "动态路由"
linkTitle: "动态路由"
weight: 10
date: 2024-06-18
description: >
  动态路由: 用 Virtual Service 和 Destination Rule 设置路由规则
---


动态路由: 用 Virtual Service 和 Destination Rule 设置路由规则


## 默认情况

没有为 bookinfo 的 app 定义 Virtual Service，只有 bookinfo 的 gateway 有 Virtual Service （这个我们在后面的网关一节中展开讲）。

```bash
k get virtualservices.networking.istio.io -n default                 
NAME       GATEWAYS               HOSTS   AGE
bookinfo   ["bookinfo-gateway"]   ["*"]   14h
```

k8s dashboard 也可以看到相关的信息，点击 "自定义资源" -> "Virtual Service" 进去。

## 演示

### 只访问 reviews v1

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

输出为:

```bash
virtualservice.networking.istio.io/productpage created
virtualservice.networking.istio.io/reviews created
virtualservice.networking.istio.io/ratings created
virtualservice.networking.istio.io/details created
```

productpage virtualservice:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
```

reviews virtualservice:

```yaml
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
        subset: v1
```

ratings virtualservice:

```yaml
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
```

details virtualservice:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
```

这里 destination / host / subset 指的是什么？

继续创建 destination rule：

```bash
k apply -f samples/bookinfo/networking/destination-rule-all.yaml
```

输出为：

```yaml
destinationrule.networking.istio.io/productpage created
destinationrule.networking.istio.io/reviews created
destinationrule.networking.istio.io/ratings created
destinationrule.networking.istio.io/details created
```

productpage DestinationRule:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
```

reviews DestinationRule:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

ratings DestinationRule:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v2-mysql
    labels:
      version: v2-mysql
  - name: v2-mysql-vm
    labels:
      version: v2-mysql-vm
```

details DestinationRule:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

查看 productpage pod 信息：

```bash
k get pods -n default                                       
NAME                             READY   STATUS    RESTARTS      AGE
details-v1-cf74bb974-6w2wq       2/2     Running   2 (15h ago)   15h
productpage-v1-87d54dd59-mdfs6   2/2     Running   2 (15h ago)   15h
ratings-v1-7c4bbf97db-gb6rx      2/2     Running   2 (15h ago)   15h
reviews-v1-5fd6d4f8f8-rvckm      2/2     Running   2 (15h ago)   15h
reviews-v2-6f9b55c5db-sxtv8      2/2     Running   2 (15h ago)   15h
reviews-v3-7d99fd7978-v952v      2/2     Running   2 (15h ago)   15h
```

```bash
k get pods -n default productpage-v1-87d54dd59-mdfs6 -o yaml 
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
  labels:
    app: productpage
    pod-template-hash: 87d54dd59
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: productpage
    service.istio.io/canonical-revision: v1
    version: v1
  name: productpage-v1-87d54dd59-mdfs6
  namespace: default
  ownerReferences:
  ......
```

看看复杂的 reviews 的 pod：


```bash
k get pods -n default reviews-v1-5fd6d4f8f8-rvckm -o yaml
k get pods -n default reviews-v2-6f9b55c5db-sxtv8 -o yaml
k get pods -n default reviews-v3-7d99fd7978-v952v -o yaml
```

```yaml
kind: Pod
metadata:
  labels:
    app: reviews
    pod-template-hash: 5fd6d4f8f8
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: reviews
    service.istio.io/canonical-revision: v1
    version: v1
  name: reviews-v1-5fd6d4f8f8-rvckm
  namespace: default
  ......
```

```yaml
kind: Pod
metadata:
  labels:
    app: reviews
    pod-template-hash: 5fd6d4f8f8
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: reviews
    service.istio.io/canonical-revision: v1
    version: v1
  name: reviews-v1-5fd6d4f8f8-rvckm
  namespace: default
  ......
```

```yaml
kind: Pod
metadata:
  labels:
    app: reviews
    pod-template-hash: 6f9b55c5db
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: reviews
    service.istio.io/canonical-revision: v2
    version: v2
  name: reviews-v2-6f9b55c5db-sxtv8
  namespace: default
  ......
```

```yaml
kind: Pod
metadata:
  labels:
    app: reviews
    pod-template-hash: 7d99fd7978
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: reviews
    service.istio.io/canonical-revision: v3
    version: v3
  name: reviews-v3-7d99fd7978-v952v
  namespace: default
  ......
```

### 只访问 reviews v2 / v3

```bash
k edit virtualservices.networking.istio.io -n default reviews
```

将 destilation 从 v1 修改为 v2，以指向  destilation rule v2: 

```yaml
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
        subset: v1    // 修改为 v2 或者 v3 
```

### 根据用户ID进行路由

virtual-service-reviews-test-v2.yaml 中定义的 reviews VirtualService：


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

让其生效，替代之前的 reviews VirtualService：

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

访问 http://192.168.0.101:30893/productpage 页面：

- 不登录：review v1
- 用 jason 登录：review v2
- 用 jason 之外的其他用户吗登录：review v1

## 清理

```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
```