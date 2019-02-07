---
date: 2018-09-29T20:00:00+08:00
title: 运行bookinfo示例
weight: 209
menu:
  main:
    parent: "installation"
description : "运行bookinfo示例"
---

在Istio安装完成之后，为了检验istio，或者学习使用istio，可以尝试先运行istio自带的示例应用bookinfo。

### 部署bookinfo

参考官方文档：

- https://istio.io/docs/examples/bookinfo/

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 验证
kubectl get services
kubectl get pods
```

### 检验ingress ip 和 端口

安装 bookinfo 的 ingress gateway：

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# 确认
$ kubectl get gateway
NAME                              AGE
bookinfo-gateway                  31m
httpbin-gateway                   19m
```

遵循指南建议，设置 `INGRESS_HOST` 和 `INGRESS_PORT` 变量以便访问 gateway。

具体做法见下一节，在完成之后回到这里继续。设置 GATEWAY_URL：

```bash
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

验证 bookinfo 应用是否运行：

```bash
$ curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage
200
```

可以通过浏览器访问地址 `http://$GATEWAY_URL/productpage` 

> 备注：GATEWAY_URL通常是 192.168.0.10:31380 这样的地址

### 检验ingress ip 和 端口

https://istio.io/docs/tasks/traffic-management/ingress/#determining-the-ingress-ip-and-ports

查看istio-ingressgateway：

```bash
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                   AGE
istio-ingressgateway   LoadBalancer   10.109.191.143   <pending>     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:30556/TCP,8060:31410/TCP,853:32282/TCP,15030:32127/TCP,15031:30924/TCP   52d
```

EXTERNAL-IP 为 <pending>，说明没有外部负载均衡器。跳到 "Determining the ingress IP and ports when using a node port"

设置 ingress 端口：

```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

设置 ingress ip （Other environments）：

```bash
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')
```

> 备注: 返回上一节去设置 GATEWAY_URL 



