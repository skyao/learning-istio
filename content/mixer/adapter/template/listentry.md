# listentry模板

 `listentry` 模板旨在使用 list 适配器执行列表检查操作。

## 模板配置

listentry模板用于验证列表中是否存在字符串。

在编写配置时，与此模板关联的字段的值可以是文字或表达式。请注意，如果字段的数据类型不是`istio.policy.v1beta1.Value`，则表达式的推断类型必须与该字段的数据类型匹配。

| 字段  | 类型   | 描述                       |
| ----- | ------ | -------------------------- |
| value | string | 指定要在列表中验证的条目。 |

配置示例:

```yaml
apiVersion: "config.istio.io/v1alpha2"
kind: listentry
metadata:
  name: appversion # 这个name就是下面所说的实例的name
  namespace: istio-system
spec:
  value: source.labels["version"]
```

在这个例子中，`spec/value` 被设置为 `source.labels["version"]`，这个表达式表示被检查的对象是 source 的 version label。

## Instance

listentry 的实例go代码为：

```go
type Instance struct {
	// 实例的Name，在配置中指定
	Name string

	// 指定要在列表中验证的条目
	Value string
}
```

## Handler

handler的go代码为：

```go
// 如果adapter想处理和'listentry'模板关联的数据，则adaper的代码必须实现这个Handler
//
// 在请求时刻，Mixer使用此接口调用adapter以发送创建的instance到adapter。
// Adapter接收传入的实例并执行他们所需的任务来实现其主要功能。
//
// 每个实例的名称可以在配置时刻通过'SetListEntryTypes'方法用作提供给adapter的类型映射中的键。
// 与实例关联的这些类型描述实例的形状。
type Handler interface {
	adapter.Handler

	// HandleListEntry方法在请求时刻被Mixer调用来传递instance到adapter
	HandleListEntry(context.Context, *Instance) (adapter.CheckResult, error)
}
```

## Handler Builder

handler builder的go代码为：

```go
type HandlerBuilder interface {
	adapter.HandlerBuilder

    // Mixer调用SetListEntryTypes为特定于adapter的instance传递模板特定的Type信息
    // adapter可能会在运行时收到这些实例。
    // 类型信息描述实例的形状。
	SetListEntryTypes(map[string]*Type /*Instance name -> Type*/)
}
```



## 后端服务的实现

lestentry adapter最终还是要调用到某个后端列表服务的(除非是纯本地，即配置中间文件中的provider_url设置为空)，因此以.proto文件的形式定义了后端服务实现的接口，文件位置`istio/mixer/template/listentry/template_handler_service.proto`:

```protobuf
service HandleListEntryService {
    // HandleListEntry 在请求时刻被mixer调用来发送'listentry'实例到后端
    rpc HandleListEntry(HandleListEntryRequest) returns (istio.mixer.adapter.model.v1beta1.CheckResult);
}
```



## 模板工作过程

1. 请求在sidecar中被提取为属性Atrribute[]，其中包含`source.labels["version"]`
2. 这些属性被传递到mixer，通过使用上述 listentry 模板来映射，Atrribute[] 中的 source.labels["version"] 被拿出来保存在 instance 中