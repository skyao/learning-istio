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

在 istio 的 [github release 页面](https://github.com/istio/istio/releases/) 找到需要的istio版本，下载并解压缩：

```bash
curl -L https://github.com/istio/istio/releases/download/1.1.8/istio-1.1.8-linux.tar.gz | tar xz
mv istio-1.1.8/ $HOME/work/soft/istio/
```

修改`~/.bashrc`，加入以下内容：

```bash
# istio
export PATH="$PATH:/home/sky/work/soft/istio/bin"
```

执行`source ~/.bashrc`载入profile。

```bash
$ istioctl version
Version: 1.0.8
GitRevision: 11b640bb11593138a904f81e572d40e5e70b089b
User: root@d76cad0a-8935-11e9-887b-0a580a2c0403
Hub: docker.io/istio
GolangVersion: go1.10.4
BuildStatus: Clean
```

## 安装istio核心

先安装istio的CRD，日常验证用最简单的方式安装所有的crd：

```bash
# 1.1版本之前
kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml

# 1.1版本之后
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
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

> 备注：如果遇到 mixer 的pod总是 crash，可以检查是不是 coredns 在CrashLoopBackOff。


