---
title: "部署bookinfo"
linkTitle: "部署bookinfo"
weight: 20
date: 2024-06-18
description: >
  使用 istioctl 安装 Istio
---

## bookinfo 案例应用

### 部署

部署 bookinfo 案例应用：

```bash
cd ~/work/soft/istio
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

输出为：

```bash
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

验证一下：

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

输出为:

```html
<title>Simple Bookstore App</title
```



### 外部访问

部署 ingress gateway 以方便从外部访问 bookinfo 应用：

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

部署完成后查看 istio-ingressgateway 服务的情况：

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

输出为：

```bash
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.102.56.34   <pending>     15021:30900/TCP,80:30893/TCP,443:31534/TCP,31400:30798/TCP,15443:31954/TCP   18m
```

简单起见我们用 node port 来访问 istio-ingressgateway

```bash
k edit service istio-ingressgateway -n istio-system
```

将默认的 ` type: LoadBalancer` 改成 ` type: NodePort`

然后用浏览器访问如下地址：

http://192.168.0.101:30893/productpage

