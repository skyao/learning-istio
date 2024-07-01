---
title: "流量转移"
linkTitle: "流量转移"
weight: 20
date: 2024-06-18
description: >
  流量转移: 灰度发布是如何实现的?
---

参考官方文档：

https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/

### 演示1: 所有流量到 v1


```bash
k apply -f samples/bookinfo/networking/destination-rule-all.yaml
k apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### 演示2: 50%流量到 v1，50%流量到 v3

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

virtual-service-reviews-50-v3.yaml 的内容：

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
      weight: 0
    - destination:
        host: reviews
        subset: v3
      weight: 100
```

### 用kiali查看流量

请求 1000 次以积累数据：

```bash
for i in $(seq 1 1000); do curl -s -o /dev/null "http://192.168.6.224:30678/productpage"; done
```

打开 kiali 查看

- namespace： default
- traffic： 只勾选 http / request
- versioned app graph
- display： 勾选 traffic distribution 和 traffic rate 


### 演示3: 完整的灰度过程（0%/1%/10%/50%/100%）

更深刻的感受灰度的过程，从 0%/1%/10%/50%/100% 一步一步灰度过来：

```bash
k edit virtualservices.networking.istio.io -n default reviews
```

依次修改百分比为 0%/1%/10%/50%/100%，每次执行请求 1000 次以积累数据：

```bash
for i in $(seq 1 1000); do curl -s -o /dev/null "http://192.168.6.224:30678/productpage"; done
```

## 清理

```bash
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
```

