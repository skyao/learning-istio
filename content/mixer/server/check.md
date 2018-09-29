# Check主流程

## 流程入口

Mixer 的 check 流程的入口在 grpcServer 的 check()方法：

> 备注：`istio/mixer/pkg/api/grpcServer.go` 

```go
// Check is the entry point for the external Check method
func (s *grpcServer) Check(legacyCtx legacyContext.Context, req *mixerpb.CheckRequest) (*mixerpb.CheckResponse, error) {

	// bag around the input proto that keeps track of reference attributes
	protoBag := attribute.NewProtoBag(&req.Attributes, s.globalDict, s.globalWordList)

	// This holds the output state of preprocess operations
	checkBag := attribute.GetMutableBag(protoBag)

	resp, err := s.check(legacyCtx, req, protoBag, checkBag)

	return resp, err
}
```

继续check()方法，去除细节代码，只看主流程：

```go
// 调用Preprocess()，前置处理
// 比如ATRRIBUTE_GENERATOR类型的adaper就会在这个时刻工作
s.dispatcher.Preprocess(legacyCtx, protoBag, checkBag);

// 调用dispatcher.check()
cr, err := s.dispatcher.Check(legacyCtx, checkBag)

if status.IsOK(resp.Precondition.Status) && len(req.Quotas) > 0 {
    // 如果check通过，再继续做quota
    qr, err = s.dispatcher.Quota(legacyCtx, checkBag, qma)
}

return resp, nil
```

##Dispatch细节

dispatcher.Check()的实现代码在 `istio/mixer/pkg/runtime/dispatcher/dispatcher.go` 中，去掉session相关细节代码，只看主要流程：

```go
func (d *Impl) Check(ctx context.Context, bag attribute.Bag) (adapter.CheckResult, error) {
	var r adapter.CheckResult
	err := s.dispatch()
	return r, err
}
```

dispatch函数非常长，去除细节，只看主要流程：

```go
func (s *session) dispatch() error {
    ......
    for _, destination := range destinations.Entries() {
        for _, group := range destination.InstanceGroups {
            for j, input := range group.Builders {
                instance, err = input.Builder(s.bag);
                state = s.impl.getDispatchState(ctx, destination)
                s.dispatchToHandler(state)
            }
        }
    }
}
```

这里面有三层for循环，逐个来看。

###循环一：destination

第一层循环是对 destinations 进行循环，这个destinations 来自Routes.GetDestinations()方法，即通过参数 namespace 和 template 的品类(品类指check/report/quota等)来获取destinations：

> 备注: 代码在`istio/mixer/pkg/runtime/routing/table.go`

```go
// GetDestinations 返回给定template 品类和给定命名空间的 destination (handler)的集合
func (t *Table) GetDestinations(variety tpb.TemplateVariety, namespace string) *NamespaceTable {
	destinations, ok := t.entries[variety]
	destinationSet := destinations.entries[namespace]
	return destinationSet
}
```

结合Table结构体的结构看上面代码的过程：

```go
type Table struct {
    // 以template品类为key的map，这是第一层map
	entries map[tpb.TemplateVariety]*varietyTable
}

type varietyTable struct {
    // 以namespace为key的map，这是第二层map
	entries map[string]*NamespaceTable
}
```

GetDestinations()方法中通过品类和namespace从Table结构中拿到了保存的NamespaceTable：

```go
type NamespaceTable struct {
	entries []*Destination
}
```

这样第一层循环就好理解了：通过variety和namespace获取保存在路由表中的destination结构，然后游历：

```go
destinations := s.rc.Routes.GetDestinations(s.variety, namespace)
for _, destination := range destinations.Entries() {
}
```

### 循环二：InstanceGroup

第二层循环，是对 Destination 的 InstanceGroups 做循环：

```go
for _, group := range destination.InstanceGroups {
}
```

看 Destination 的结构：

```go
type Destination struct {
	// 要调用的handler
	Handler adapter.Handler

	// handler的Template
	Template *template.Info

    // 适用于handler的InstanceGroups
	InstanceGroups []*InstanceGroup
}
```

注意这时我们已经拿到了handler，template，和InstanceGroup数组。

循环三：

第三层循环，是对 InstanceGroup 的 Builders 做循环：

```go
for j, input := range group.Builders {
}
```

看 InstanceGroup 的结构：

```go
type InstanceGroup struct {
	// Condition for applying this instance group.
	Condition compiled.Expression

	// Builders for the instances in this group for each instance that should be applied.
	Builders []NamedBuilder
}
```

在三层循环中，终于开始处理了

```go
// 根据属性build出来instance
instance, err = input.Builder(s.bag);
state = s.impl.getDispatchState(ctx, destination)
// 如名字所示，转发给handler
s.dispatchToHandler(state)
```

## 调用handler

dispatchToHandler()方法的代码在 `istio/mixer/pkg/runtime/dispatcher/session.go` 文件中：

```go
func (s *session) dispatchToHandler(ds *dispatchState) {
	s.activeDispatches++
	ds.session = s
	s.impl.gp.ScheduleWork(ds.invokeHandler, nil)
}
```

`s.impl.gp` 是一个 `pool.GoroutinePool`，执行传入的WorkFunc，ds.invokeHandler 的定义在 `istio/mixer/pkg/runtime/dispatcher/dispatchstate.go `：

```go
func (ds *dispatchState) invokeHandler(interface{}) {
    // 根据模板品类调用不同的函数ds.destination.Template.×××
	switch ds.destination.Template.Variety {
	case tpb.TEMPLATE_VARIETY_ATTRIBUTE_GENERATOR:
		ds.outputBag, ds.err = ds.destination.Template.DispatchGenAttrs(
			ctx, ds.destination.Handler, ds.instances[0], ds.inputBag, ds.mapper)

	case tpb.TEMPLATE_VARIETY_CHECK:
		ds.checkResult, ds.err = ds.destination.Template.DispatchCheck(
			ctx, ds.destination.Handler, ds.instances[0])

	case tpb.TEMPLATE_VARIETY_REPORT:
		ds.err = ds.destination.Template.DispatchReport(
			ctx, ds.destination.Handler, ds.instances)

	case tpb.TEMPLATE_VARIETY_QUOTA:
		ds.quotaResult, ds.err = ds.destination.Template.DispatchQuota(
			ctx, ds.destination.Handler, ds.instances[0], ds.quotaArgs)
}
```

`ds.destination.Template` 的类型是 `template.Info`：

```go
type Info struct {
    DispatchReport   DispatchReportFn
    DispatchCheck    DispatchCheckFn
    DispatchQuota    DispatchQuotaFn
    DispatchGenAttrs DispatchGenerateAttributesFn
}
```

这几个Fn的设置在文件 `istio/mixer/template/template.gen.go` 中，这个一个自动生成的类，SupportedTmplInfo 保存有每个 handler 的信息，以 listentry 和 DispatchCheck 为例：

```go
var (
	SupportedTmplInfo = map[string]template.Info{
        ......
        listentry.TemplateName: {
			Name:               listentry.TemplateName,
			Impl:               "listentry",
			DispatchCheck: func(ctx context.Context, handler adapter.Handler, inst interface{}) (adapter.CheckResult, error) {
				instance := inst.(*listentry.Instance)
				return handler.(listentry.Handler).HandleListEntry(ctx, instance)
			},
            ......
    }
)
```

最后调用到这个 listentry.Handler 的 HandleListEntry()方法。

