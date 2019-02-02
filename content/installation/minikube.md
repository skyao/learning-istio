---
date: 2018-09-29T20:00:00+08:00
title: minikube安装
weight: 202
menu:
  main:
    parent: "installation"
description : "使用minikube安装istio"
---

## 准备Minikube

安装最新版本的 Minikube。

详细见[Minikube安装](https://skyao.gitbooks.io/learning-kubernetes/installation/minikube.html)

启动minikube的命令：

```bash
minikube start
```

如果需要翻墙：

```bash
minikube start --docker-env http_proxy=http://192.168.31.152:8123 --docker-env https_proxy=http://192.168.31.152:8123 --docker-env no_proxy=localhost,127.0.0.1,::1,192.168.31.0/24,192.168.99.0/24
```

## 准备istio

参考：

- https://istio.io/docs/setup/kubernetes/download-release/
- https://istio.io/docs/setup/kubernetes/quick-start/

在linux上执行以下命令：

```bash
curl -L https://git.io/getLatestIstio | sh -
sudo mv istio-1.0.5/ $HOME/work/soft/istio/
```

修改`~/.bashrc`，加入以下内容：

```bash
# istio
export PATH="$PATH:/home/sky/work/soft/istio/bin"
```

执行`source ~/.bashrc`载入profile。

```bash
$ istioctl version
Version: 1.0.5
GitRevision: c1707e45e71c75d74bf3a5dec8c7086f32f32fad
User: root@6f6ea1061f2b
Hub: docker.io/istio
GolangVersion: go1.10.4
BuildStatus: Clean
```

## 安装istio核心

先安装istio的CRD：

```bash
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
```

简单起见，不做双向认证。

```bash
kubectl apply -f install/kubernetes/istio-demo.yaml
```

验证安装：

```bash
kubectl get svc -n istio-system
kubectl get pods -n istio-system
```


