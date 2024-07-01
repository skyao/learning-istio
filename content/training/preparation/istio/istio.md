---
title: "使用 istioctl 安装 Istio"
linkTitle: "安装istio"
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
istioctl version
```

输出为：

```bash
$ istioctl version
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


