# Mixer Adapter

Mixer Adapter的介绍，请见官方博客文章：

- [Mixer Adapter Model](https://istio.io/blog/2017/adapter-model/)
- [Mixer Adapter Model(中文翻译)](http://istio.doczh.cn/blog/2017/adapter-model.html)

## 为什么需要Adapter？

摘抄Mixer Adapter Model中的描述：

> 各种基础设施都提供了用于支持服务构建的功能，例如访问控制、遥测、配额、计费等等。传统服务会直接和这些后端系统打交道，和后端紧密耦合，并集成其中的个性化语义以及用法。
> 
> Mixer 服务构成了基础设施和 Istio 之间的抽象层。Istio 组件和运行于 Service Mesh 中的服务，借助 Mixer 的能力就能在不直接访问后端接口的情况下和这些后端进行交互。
> 
> Mixer 除了作为应用和基础设施之间的隔离层之外，操作者还可以借助 Mixer 的中介模型，注入或者操作应用和后端之间的策略，例如哪些数据需要报告给哪些后端、哪些后端提供认证等。
> 
> 每种后端都有各自不同的接口和操作方式，因此 Mixer 需要有代码来支持这种差异，我们称这些内容为[适配器](https://github.com/istio/istio/wiki/Mixer-Adapter-Dev-Guide)。
> 
> 适配器是 Go 包的形式存在的，直接链接到 Mixer 二进制中。如果缺省适配器无法满足特定的用例，创建自己的适配器也是比较简单的。

Adapter的哲学:

> 本质上 Mixer 这个模块就是用来处理属性和路由的。代理把[属性](http://istio.doczh.cn/docs/concepts/policy-and-control/attributes.html)作为前置检查和遥测报告的一部分发送出来，转换为对适配器的一系列调用。运维人员提供了用于描述如何将属性转换为适配器指令的配置。

![](http://istio.doczh.cn/docs/concepts/policy-and-control/img/mixer-config/machine.svg)

> 配置是一个复杂的任务。有证据表明，绝大多数的服务中断都来自于配置错误。为了解决这一问题，Mixer 加入了很多限制来避免错误。例如在配置中使用强类型，以此保障在任何上下文中都只能使用有意义的属性或表达式。

## Adapter信息

每个adapter的GetInfo()方法返回该adapter实现关联的信息，如：

```go
func GetInfo() adapter.Info {
	return adapter.Info{
		Name:        "bypass", // Adapter名称
		Impl:        "istio.io/istio/mixer/adapter/bypass", // Adapter实现
		Description: "Calls gRPC backends via the inline adapter model (useful for testing)", // Adapter描述
		SupportedTemplates: []string{  // Adapter支持的模板
			checknothing.TemplateName,
			reportnothing.TemplateName,
			metric.TemplateName,
			quota.TemplateName,
		},
		DefaultConfig: &config.Params{},
		NewBuilder:    func() adapter.HandlerBuilder { return &builder{} },
	}
}
```



## 已有的Adapter

| Adapter名称       | Adapter描述          | 支持的模板                                                   | 模板品类                     |
| ----------------- | -------------------- | ------------------------------------------------------------ | ---------------------------- |
| bypass            |                      | checknothing, reportnothing, metric, quota                   | CHECK, REPORT, REPORT, QUOTA |
| circonus          |                      | metric                                                       | REPORT                       |
| cloudwatch        |                      | metric                                                       | REPORT                       |
| denier            |                      | checknothing, listentry, quota                               | CHECK, QUOTA                 |
| dogstatsd         |                      | metric                                                       | REPORT                       |
| fluentd           |                      | logentry                                                     | REPORT                       |
| kubernetesenv     |                      | kubernetesenv                                                | ATTRIBUTE_GENERATOR          |
| listchecker(list) |                      | listentry                                                    | CHECK                        |
| memquota          |                      | quota                                                        | QUOTA                        |
| noop              | 空操作，支持所有模板 | authorization, checknothing, reportnothing, listentry, logentry, metric, quota, tracespan | 所有                         |
| opa               |                      | authorization                                                | CHECK                        |
| prometheus        |                      | metric                                                       | REPORT                       |
| rbac              |                      | authorization                                                | CHECK                        |
| redisquota        |                      | quota                                                        | QUOTA                        |
| servicecontrol    |                      | apikey, servicecontrolreport, quota                          | CHECK, REPORT, QUOTA         |
| solarwinds        |                      | metric, logentry                                             | REPORT                       |
| stackdriver       |                      | metric, logentry, tracespan                                  | REPORT                       |
| statsd            |                      | metric                                                       | REPORT                       |
| stdio             |                      | logentry, metric                                             | REPORT                       |



