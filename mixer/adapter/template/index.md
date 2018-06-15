# Mixer Adapter 模板

## 什么是Adapter 模板？

> 请求到达 Mesh 中的服务时，一般会发生两次对 Mixer 的调用，一次是前置检查，一次是遥测报告。每一次这种调用，Mixer 都需要调用一个或更多的适配器。不同的适配器需要不同的数据块作为输入来进行处理。例如日志适配器需要日志输入，指标适配器需要指标输入，认证适配器需要凭据输入。适配器在请求时消费的数据就是由 Mixer 的 [Template](https://istio.io/docs/reference/config/mixer/template/) 来描述的。
>
> 每个 Template 对应一个 protobuf 消息。在运行时Templates描述了一系列的发送给一或多个适配器的数据。适配器和 Template 是多对多的关系，其对应关系由开发者决定。
>
> Metric和Logentry是最重要的两个 Template，分别用于描述工作负载的指标和日志。

### 模板品类

每个模板都有一个属性 `template_variety`，用来标明模板适用于什么操作。

定义于文件 `istio/vendor/istio.io/api/mixer/adapter/model/v1beta1/extensions.pb.go`：

- TEMPLATE_VARIETY_CHECK
- TEMPLATE_VARIETY_REPORT
- TEMPLATE_VARIETY_QUOTA
- TEMPLATE_VARIETY_ATTRIBUTE_GENERATOR

## 现有模板

模板的实现代码在`istio/mixer/template`，当前现在模板如下：

| 名称                 | 描述                                              | 备注                                      | 品类                |
| -------------------- | ------------------------------------------------- | ----------------------------------------- | ------------------- |
| apikey               | 单个API key                                       | 用于认证检查                              | CHECK               |
| authorization        | 定义了在Istio内执行策略强化的参数                 | 主要关注启动Mixer                         | CHECK               |
| checkNothing         | 空数据块，用于支持check的adapter，不需要任何参数  | 主要用于测试                              | CHECK               |
| listentry            | 用于验证列表中是否存在字符串                      | 设计用来使用 list adapter实现名单检查操作 | CHECK               |
| logentry             | 表示日志中的单个条目                              |                                           | REPORT              |
| metric               | 表示报告的单个数据片段                            | 设计用来描述要发送到监控后端的运行时度量  | REPORT              |
| quota                | 检查配额使用的数据块                              |                                           | QUOTA               |
| reportNothing        | 空数据块，用于支持report的adapter，不需要任何参数 | 主要用于测试                              | REPORT              |
| traceSpan            | 表示分布式追踪中的单个span                        |                                           | REPORT              |
| kubernetesenv        | 获取k8s环境消息                                   | 用于生成属性                              | ATTRIBUTE_GENERATOR |
| servicecontrolreport |                                                   | 只用于service control report              | REPORT              |

备注：有几个模板的实现代码位置特殊，直接写在Adapter实现的目录下。

1. kubernetesenv: 代码在`istio/mixer/adapter/kubernetesenv/template`
2. servicecontrolreport: 代码在`istio/mixer/adapter/servicecontrol/template`