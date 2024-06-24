---
title: "使用 istioctl 安装 Istio"
linkTitle: "istioctl"
weight: 10
date: 2024-06-18
description: >
  使用 istioctl 安装 Istio
---


参考：

- https://istio.io/latest/docs/setup/install/istioctl/

## 下载 Istio


```bash
curl -L https://istio.io/downloadIstio | sh -
```

输出为：

```bash
curl -L https://istio.io/downloadIstio | sh -
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   101  100   101    0     0    405      0 --:--:-- --:--:-- --:--:--   407
100  4899  100  4899    0     0   4660      0  0:00:01  0:00:01 --:--:-- 18348

Downloading istio-1.22.1 from https://github.com/istio/istio/releases/download/1.22.1/istio-1.22.1-linux-amd64.tar.gz ...

Istio 1.22.1 Download Complete!

Istio has been successfully downloaded into the istio-1.22.1 folder on your system.

Next Steps:
See https://istio.io/latest/docs/setup/install/ to add Istio to your Kubernetes cluster.

To configure the istioctl client tool for your workstation,
add the /home/sky/istio-1.22.1/bin directory to your environment path variable with:
	 export PATH="$PATH:/home/sky/istio-1.22.1/bin"

Begin the Istio pre-installation check by running:
	 istioctl x precheck 

Need more information? Visit https://istio.io/latest/docs/setup/install/
```

移动到 `~/work/soft/istio` 目录：

```bash
mv istio-1.22.1 ~/work/soft/istio
```

加入到 PATH 路径：

```bash
vi ~/.zshrc
```

增加内容：

```bash
# istio
export PATH=/home/sky/work/soft/istio/bin:$GOPATH/bin:$PATH
```

载入：

```bash
source ~/.zshrc 
```

验证：

```bash
istio istioctl version
```

输出为：

```bash
$ istio istioctl version
no ready Istio pods in "istio-system"
```

## 安装 Istio

先用 demo 验证一下安装：

```bash
istioctl install --set profile=demo -y
```

输出为：

```bash
$ istio istioctl install --set profile=demo -y
✔ Istio core installed                                      
✔ Istiod installed                       
✔ Ingress gateways installed                        
✔ Egress gateways installed                          
✔ Installation complete

Made this installation the default for injection and validation.
```

自动注入 sidecar：

```bash
kubectl label namespace default istio-injection=enabled
```

### 验证安装

查看当前 istio-system 下的 service：

```bash
k get services -n istio-system 
```

输出为：

```bash
$ k get services -n istio-system 
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.103.105.85    <none>        80/TCP,443/TCP                                                               2m59s
istio-ingressgateway   LoadBalancer   10.102.56.34     <pending>     15021:30900/TCP,80:30893/TCP,443:31534/TCP,31400:30798/TCP,15443:31954/TCP   2m59s
istiod                 ClusterIP      10.105.153.105   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        3m11s
```

查看当前 istio-system 下的 service：

```bash
k get pod -n istio-system
```

输出为：

```bash
$ k get pod -n istio-system
NAME                  READY  STATUS  RESTARTS  AGE
istio-egressgateway-b8b9b64f4-gphx5   1/1   Running  0     4m21s
istio-ingressgateway-6b7c788c74-vr6m9  1/1   Running  0     4m21s
istiod-64d8d5769b-jb9dn         1/1   Running  0     4m33s
```



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



## 安装 kiali

### 安装

安装 kiali 和其他附件（包括 grafana / zipkin / prometheus）  ：

```bash
istio kubectl apply -f samples/addons
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

就可以在 kiali 中看到数据了。
