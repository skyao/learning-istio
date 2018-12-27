---
date: 2018-09-29T20:00:00+08:00
title: Istio Authentication CRD
weight: 2020
description : "介绍istio的authentication CRD"
---

authentication.istio.io

相关资料：

- 官方资料的配置参考： https://preliminary.istio.io/docs/reference/config/istio.authentication.v1alpha1/#Policy
- 代码在 `istio/api` 仓库下的 `authentication/v1alpha1` 目录

### Policy

策略定义在工作负载上可以接受的身份验证方法，如果验证通过，哪个方法/证书将设置为请求主体(request principal)（即 `request.auth.principal` 属性）。

验证策略由两部分验证组成： 

- peer：验证调用者服务证书。这部分将设置 `source.user`（peer identity）。 
- origin：验证原始证书。此部分将设置 `request.auth.user`（origin identity），以及`request.auth.presenter`，`request.auth.audiences`和 raw claim 等其他属性。

请注意，身份可以是最终用户(end-user)，服务帐户(service account)，设备(device)等。

最后但并非最不重要的是，主要绑定规则(principal binding rule)定义应将哪个身份（peer 或者 origin）用作主体。默认情况下，使用peer。

### Mesh Policy

TBD：没有找到介绍资料

`pilot/pkg/config/kube/crd/types.go` 中定义如下：

```go
// MeshPolicy is the generic Kubernetes API object wrapper
type MeshPolicy struct {
	meta_v1.TypeMeta   `json:",inline"`
	meta_v1.ObjectMeta `json:"metadata"`
	Spec               map[string]interface{} `json:"spec"`
}
```

`install/kubernetes/helm/istio/templates/crds.yaml`:

```yam
kind: CustomResourceDefinition
apiVersion: apiextensions.k8s.io/v1beta1
metadata:
  name: meshpolicies.authentication.istio.io
  annotations:
    "helm.sh/hook": crd-install
  labels:
    app: istio-citadel
    chart: istio
    heritage: Tiller
    release: istio
spec:
  group: authentication.istio.io
  names:
    kind: MeshPolicy
    listKind: MeshPolicyList
    plural: meshpolicies
    singular: meshpolicy
    categories:
    - istio-io
    - authentication-istio-io
  scope: Cluster
  version: v1alpha1
```

