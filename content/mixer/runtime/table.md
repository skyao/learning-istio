# Table

## Table结构

Table是主路由表。用于查找应该被调用的handler集合，以及实例构建器和匹配条件。

```go
type Table struct {
    // 此表的ID。基于配置快照(snapshot)ID。ID在Mixer实例的生命周期内是唯一的。
	id int64

	// 用variety分组的namespaceTables
    // 也就是说，map的key是variety，value是varietyTable
	entries map[tpb.TemplateVariety]*varietyTable

	debugInfo *tableDebugInfo
}
```

TBD: 配置快照是什么？代码有撇到，后面细看。

varietyTable的结构，还是有一个map：

```go
// varietyTable包含给定模板类型的destination集合。
// 它包含从命名空间到destination的扁平列表的映射。
// 它还包含defaultSet，在找不到命名空间特定的destination条目时返回
type varietyTable struct {
    // 用 namespace 分组的 destination。
    // 也就是说，map的key是namespace，value是NamespaceTable
    // 也包括来自默认命名空间的destination
	entries map[string]*NamespaceTable

	// 默认 namespace 的 destinations
	defaultSet *NamespaceTable
}
```

NamespaceTable的结构：

```go
// NamespaceTable 包含对应给定namespace的 destination 列表，
type NamespaceTable struct {
	entries []*Destination
}
```

总结： 这样通过 `Table[variety][namespace]` 就可以拿到 Table 中存储的对应 variety 和 namespace 相关的 Destination 了。

## Destination结构

Destination 包含目标handler和要发送的instance，按照适用于它们的条件匹配进行分组。

```go
type Destination struct {
    // 条目的id。每次table被重建时，重用id。用于调试
    // TBD: IDs are reused every time a table is recreated，看不懂，等后面继续翻代码
	id uint32

	// 要调用的Handler
	Handler adapter.Handler

	// HandlerName 是handle的名字。用于监控、日志目的
	HandlerName string

	// AdapterName 是adapter的名字。用于监控、日志目的
	AdapterName string

	// handler的Template
	Template *template.Info

    // 可能(有条件的)适用于handler的 InstanceGroups
	InstanceGroups []*InstanceGroup

    // 可以从这个条目中创建的实例的最大数量
	maxInstances int

	// FriendlyName 是这个配置好的handler条目的友好名称。用来监控日志目的。
	FriendlyName string

	// 计数器，用来追踪到adapters/handler的分配情况
	Counters DestinationCounters
}
```

maxInstances 的计算方式如下，实际是当前 condition 里面所有InstanceGroups下的Builders的总数，每个Builders可以创建一个instance，因此这个数量就是所谓"可以从这个条目中创建的实例的最大数量"：

```
func (d *Destination) recalculateMaxInstances() {
   c := 0
   for _, input := range d.InstanceGroups {
      c += len(input.Builders)
   }

   d.maxInstances = c
}
```

## Instance结构