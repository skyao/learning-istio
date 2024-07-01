---
title: "安装 kiali"
linkTitle: "安装kiali"
weight: 20
date: 2024-06-18
description: >
  使用 kiali 查看 istio
---

## 安装 kiali

### 安装

安装 kiali 和其他附件（包括 grafana / zipkin / prometheus）  ：

```bash
cd ~/work/soft/istio
kubectl apply -f samples/addons
```

输出为：

```bash
serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/loki created
configmap/loki created
configmap/loki-runtime created
service/loki-memberlist created
service/loki-headless created
service/loki created
statefulset.apps/loki created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

等待安装完成：

```bash
kubectl rollout status deployment/kiali -n istio-system
```

### 外部访问

查看 kiali 服务的情况：

```bash
kubectl get svc kiali -n istio-system
```

输出为：

```bash
$ kubectl get svc kiali -n istio-system
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
kiali   ClusterIP   10.99.136.240   <none>        20001/TCP,9090/TCP   5m8s
```

修改为 node port：

```bash
kubectl edit svc kiali -n istio-system
```

将默认的 ` type: LoadBalancer` 改成 ` type: NodePort`。之后再次查看：

```bash
kubectl get svc kiali -n istio-system 
```

可以看到 20001 端口已经被映射到 32367 端口：

```bash
kubectl get svc kiali -n istio-system 
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
kiali   NodePort   10.99.136.240   <none>        20001:32367/TCP,9090:32582/TCP   6m18s
```

打开另外一个终端，执行命令：

```bash
istioctl dashboard kiali
```

输出为;

```bash
istioctl dashboard kiali
http://localhost:20001/kiali
```

用浏览器访问如下地址：

http://192.168.0.101:32367

### kiali的简单使用

先给一点请求，以便产生数据：

```bash
for i in $(seq 1 100); do curl -s -o /dev/null "http://192.168.0.101:30893/productpage"; done 
```

```bash
for i in $(seq 1 100); do curl -s -o /dev/null "http://${IP}:30893/productpage"; done 
```

就可以在 kiali 中看到数据了。
