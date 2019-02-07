---
date: 2019-02-02T18:00:00+08:00
title: 搭建开发环境
weight: 209
menu:
  main:
    parent: "installation"
description : "搭建Istio开发环境"
---

参考Istio官方文档：

https://github.com/istio/istio/wiki/Dev-Guide

### 准备环境

按照官方文档的要求先准备好golang/docker/kubernetes/fpm。

同样按照官方文档的要求设置环境变量：

```bash
export GOPATH=/home/sky/work/soft/go
export PATH=$GOPATH/bin:$PATH
export ISTIO=$GOPATH/src/istio.io
export HUB="docker.io/aoxiaojian"
export TAG=aoxiaojian
export GITHUB_USER=skyao
export KUBECONFIG=${HOME}/.kube/config
```

## build

在istio目录下执行make命令：

```bash
make
make docker
```

然后通过Docker images命令可以看到新build出来的docker镜像：

```bash
$ docker images
REPOSITORY                           TAG                                        IMAGE ID            CREATED              SIZE
istio/node-agent-k8s                 98c2fe55136b584f19499615a509143127ac7532   a988dfde603a        17 seconds ago       235MB
istio/kubectl                        98c2fe55136b584f19499615a509143127ac7532   eb9b7c9b5de1        41 seconds ago       360MB
istio/sidecar_injector               98c2fe55136b584f19499615a509143127ac7532   c004d72da376        About a minute ago   47.1MB
istio/galley                         98c2fe55136b584f19499615a509143127ac7532   910ba0296ef2        About a minute ago   320MB
istio/citadel                        98c2fe55136b584f19499615a509143127ac7532   606bfc1a071d        About a minute ago   51.3MB
istio/mixer_codegen                  98c2fe55136b584f19499615a509143127ac7532   9e0519ca2dfa        About a minute ago   15.8MB
istio/mixer                          98c2fe55136b584f19499615a509143127ac7532   491c656242ac        About a minute ago   71.4MB
istio/servicegraph                   98c2fe55136b584f19499615a509143127ac7532   f8c7d7fc60e5        About a minute ago   11.4MB
istio/proxy_init                     98c2fe55136b584f19499615a509143127ac7532   f8f0e35bbc4a        About a minute ago   121MB
istio/test_policybackend             98c2fe55136b584f19499615a509143127ac7532   26848d4ce62a        2 minutes ago        265MB
istio/app                            98c2fe55136b584f19499615a509143127ac7532   0bc69e99902b        2 minutes ago        316MB
istio/proxyv2                        98c2fe55136b584f19499615a509143127ac7532   0ec135749c6b        2 minutes ago        370MB
istio/proxytproxy                    98c2fe55136b584f19499615a509143127ac7532   df4354e32772        2 minutes ago        370MB
istio/proxy_debug                    98c2fe55136b584f19499615a509143127ac7532   da1009d4e234        2 minutes ago        932MB
istio/pilot                          98c2fe55136b584f19499615a509143127ac7532   c539ef7acb36        3 minutes ago        309MB
```





## debug

- 在本地开发阶段可以方便的debug代码，代码更新后，不用重新构建镜像，可以高效的进行debug
- debug环境开箱即用，不需要每个开发人员进行配置

在本地启动container，挂载本地的workspace，docker bash 上去进行dlv

### 使用方式

```
cd $istio_src 
# 挂载本地的src到docker，修改马上生效
docker run -it --security-opt=seccomp:unconfined -p 2345:2345  -v $(pwd):/go/src/istio.io/istio -w /go/src/istio.io/istio reg.docker.alibaba-inc.com/sofastack/meshci-1.11:0.1 bash 

# 登录后对某个UT进行debug，例如xds_test.go 的TestEnvoy func
dlv test --build-flags='pilot/pkg/proxy/envoy/v2/xds_test.go' --headless --api-version=2 --listen=0.0.0.0:2345 -- -test.run ^TestEnvoy$
```

goland通过配置

![img](images/debug.png)



