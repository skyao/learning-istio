---
date: 2019-02-02T09:00:00+08:00
title: Kubernets实现
weight: 514
menu:
  main:
    parent: "pilot-discovery"
description : "Pilot 服务注册与发现抽象模型的Kubernets实现"
---

Kubernets实现

## Controller的定义

ControllerOptions 存储Controller的可配置属性。

```go
type ControllerOptions struct {
	// Controller监控的Namespace. 如果设置为 meta_v1.NamespaceAll (""), controller 监控所有的 namespaces
	WatchedNamespace string
	ResyncPeriod     time.Duration
	DomainSuffix     string

	// ClusterID 在多集群环境中标识远程集群
	ClusterID string

	// XDSUpdater 将变更推送给 xDS server.
	XDSUpdater model.XDSUpdater

	// 在 SPIFFE identity 中使用的 TrustDomain 
	TrustDomain string

	stop chan struct{}
}
```

Controller的 struct 定义：

```go
type Controller struct {
	domainSuffix string

	client    kubernetes.Interface
	queue     Queue
	services  cacheHandler
	endpoints cacheHandler
	nodes     cacheHandler

	pods *PodCache

	// Env 由server设置，用于指向环境，以容许controller使用 env 数据并推送状态。测试中可能为null
	Env *model.Environment

	// ClusterID 在多集群环境中标识远程集群
	ClusterID string

	// XDSUpdater 将推送 EDS 变更到 ADS model.
	XDSUpdater model.XDSUpdater

	stop chan struct{}

	sync.RWMutex
	// servicesMap 存储 hostname ==> service, 用于减少对 convertService 的调用.
	servicesMap map[model.Hostname]*model.Service
	// externalNameSvcInstanceMap 存储 hostname ==> instance, 用于存储 ExternalName k8s services 的实例
	externalNameSvcInstanceMap map[model.Hostname][]*model.ServiceInstance

	// CIDR ranger based on path-compressed prefix trie
	ranger cidranger.Ranger

	// registry 的 Network name，通过 MeshNetworks 的 configmap 指定
	networkForRegistry string
}
```

重点关注这四个内容：

```go
	services  cacheHandler
	endpoints cacheHandler
	nodes     cacheHandler

	pods *PodCache
```

### cacheHandler

其中 cacheHandler 的定义是：

```go
type cacheHandler struct {
	informer cache.SharedIndexInformer
	handler  *ChainHandler
}

// ChainHandler ChainHandler按顺序应用handler
type ChainHandler struct {
	funcs []Handler
}

// Apply方法用于执行各个handler
func (ch *ChainHandler) Apply(obj interface{}, event model.Event) error {
    // 按照顺序
	for _, f := range ch.funcs {
		if err := f(obj, event); err != nil {
			return err
		}
	}
	return nil
}

// 通过Append方法将一个 handler 加入到链的最后面
func (ch *ChainHandler) Append(h Handler) {
	ch.funcs = append(ch.funcs, h)
}
```

这样 services / endpoints / nodes 三个字段就分别保存有各自的  Handler Chain。

### PodCache

PodCache是最终一致性的 pod cache：

```go
type PodCache struct {
	cacheHandler

	sync.RWMutex
	// keys 维护稳定的pod ip 到 name 的映射。
	// 这容许我们通过 pod ip 获取最新的状态。
	// 应该只包含 RUNNING 或者 PENDING 状态的pod，带有已分配的IP
	keys map[string]string

    // 对当前podcache所在的controller的引用
	c *Controller
}
```

除了同样有cacheHandler之外，还有其他几个属性。

newPodCache()方法，在cacheHandler中增加了handler，以回调 event() 方法：

```go
func newPodCache(ch cacheHandler, c *Controller) *PodCache {
	out := &PodCache{
		cacheHandler: ch,
		c:            c,
		keys:         make(map[string]string),
	}

    // 额外增加一个handler在cacheHandler中
	ch.handler.Append(func(obj interface{}, ev model.Event) error {
        // 用在调用 PodCache 的 event() 方法
		return out.event(obj, ev)
	})
	return out
}
```

event()方法是关键，这个方法会根据event的信息，来决定更新 pod cache 中 keys 字段保存的 pod ip 到 name 的索引信息，并调用 XDSUpdater：

```go
// event 更新 IP-based 索引 (podccache 的 keys 字段).
func (pc *PodCache) event(obj interface{}, ev model.Event) error {

	// 当 pod 被删除时， obj 可以是 *v1.Pod 或者 DeletionFinalStateUnknown marker item.
	pod, ok := obj.(*v1.Pod)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			return fmt.Errorf("couldn't get object from tombstone %+v", obj)
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			return fmt.Errorf("tombstone contained object that is not a pod %#v", obj)
		}
	}

    // 拿到 pod 对象之后
	ip := pod.Status.PodIP
	// PodIP will be empty when pod is just created, but before the IP is assigned
	// via UpdateStatus.

	if len(ip) > 0 {
		log.Infof("Handling event %s for pod %s in namespace %s -> %v", ev, pod.Name, pod.Namespace, ip)
		key := KeyFunc(pod.Name, pod.Namespace)
		switch ev {
		case model.EventAdd:
			switch pod.Status.Phase {
			case v1.PodPending, v1.PodRunning:
				// 添加到cache，如果pod 是 running 或者 pending
				pc.keys[ip] = key
				if pc.c != nil && pc.c.XDSUpdater != nil {
                    // 通知到 XDSUpdater
					pc.c.XDSUpdater.WorkloadUpdate(ip, pod.ObjectMeta.Labels, pod.ObjectMeta.Annotations)
				}
			}
		case model.EventUpdate:
			switch pod.Status.Phase {
			case v1.PodPending, v1.PodRunning:
				pc.keys[ip] = key
				if pc.c != nil && pc.c.XDSUpdater != nil {
					pc.c.XDSUpdater.WorkloadUpdate(ip, pod.ObjectMeta.Labels, pod.ObjectMeta.Annotations)
				}
			default:
				// delete if the pod switched to other states and is in the cache
				if pc.keys[ip] == key {
					delete(pc.keys, ip)
					if pc.c != nil && pc.c.XDSUpdater != nil {
						pc.c.XDSUpdater.WorkloadUpdate(ip, nil, nil)
					}
				}
			}
		case model.EventDelete:
			// delete only if this pod was in the cache
			if pc.keys[ip] == key {
				delete(pc.keys, ip)
				if pc.c != nil && pc.c.XDSUpdater != nil {
					pc.c.XDSUpdater.WorkloadUpdate(ip, nil, nil)
				}
			}
		}
	}
	return nil
}
```

## Controller的构建

### NewController()

NewController()方法创建一个新的 Kubernetes controller。通常是通过 bootstrap 和 multicluster 创建：

```go
func NewController(client kubernetes.Interface, options ControllerOptions) *Controller {
	// Queue 需要一个 time duration，用于在handler出错之后的重试延迟, 这里hard code 为 1 秒钟
	out := &Controller{
		domainSuffix:               options.DomainSuffix,
		client:                     client,
		queue:                      NewQueue(1 * time.Second),
		ClusterID:                  options.ClusterID,
		XDSUpdater:                 options.XDSUpdater,
		servicesMap:                make(map[model.Hostname]*model.Service),
		externalNameSvcInstanceMap: make(map[model.Hostname][]*model.ServiceInstance),
	}

    // 创建sharedInformers
	sharedInformers := informers.NewSharedInformerFactoryWithOptions(client, options.ResyncPeriod, informers.WithNamespace(options.WatchedNamespace))

    // 在分别创建 service / endpoint / node / pod 的 informer和 cacheHandler 对象
	svcInformer := sharedInformers.Core().V1().Services().Informer()
	out.services = out.createCacheHandler(svcInformer, "Services")

	epInformer := sharedInformers.Core().V1().Endpoints().Informer()
	out.endpoints = out.createEDSCacheHandler(epInformer, "Endpoints")

	nodeInformer := sharedInformers.Core().V1().Nodes().Informer()
	out.nodes = out.createCacheHandler(nodeInformer, "Nodes")

	podInformer := sharedInformers.Core().V1().Pods().Informer()
    // pod 特殊一点，还要在 cacheHandler 外面包装一个 PodCache
	out.pods = newPodCache(out.createCacheHandler(podInformer, "Pod"), out)

	return out
}
```

### createCacheHandler()

createCacheHandler()方法为特定事件注册handler。

当前实现在 queue.go 中将事件排队，并且 handler 在运行时会有一些限流。

用于 Service, Endpoint, Node 和 Pod。

```go
func (c *Controller) createCacheHandler(informer cache.SharedIndexInformer, otype string) cacheHandler {
    // 现在 handler 中加入第一个 handler，来自 c.notify
	handler := &ChainHandler{funcs: []Handler{c.notify}}

    // 然后在 informer 上添加 EventHandler
	informer.AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				k8sEvents.With(prometheus.Labels{"type": otype, "event": "add"}).Add(1)
                // push task 到 Controller.queue
				c.queue.Push(Task{handler: handler.Apply, obj: obj, event: model.EventAdd})
			},
			UpdateFunc: func(old, cur interface{}) {
				if !reflect.DeepEqual(old, cur) {
					k8sEvents.With(prometheus.Labels{"type": otype, "event": "update"}).Add(1)
					c.queue.Push(Task{handler: handler.Apply, obj: cur, event: model.EventUpdate})
				} else {
					k8sEvents.With(prometheus.Labels{"type": otype, "event": "updateSame"}).Add(1)
				}
			},
			DeleteFunc: func(obj interface{}) {
				k8sEvents.With(prometheus.Labels{"type": otype, "event": "delete"}).Add(1)
				c.queue.Push(Task{handler: handler.Apply, obj: obj, event: model.EventDelete})
			},
		})

	return cacheHandler{informer: informer, handler: handler}
}
```

其中 Controller.notify 是 handler chain 中的第一个 handler，如果这个方法返回error，会导致整个 chain 的重复执行：

```go
func (c *Controller) notify(obj interface{}, event model.Event) error {
    // 检查 Controller 的状态，是否已经同步完成
	if !c.HasSynced() {
		return errors.New("waiting till full synchronization")
	}
	return nil
}

// HasSynced 在初始化状态同步完成之后返回true
func (c *Controller) HasSynced() bool {
    // 需要检查四个informer的同步状态
	if !c.services.informer.HasSynced() ||
		!c.endpoints.informer.HasSynced() ||
		!c.pods.informer.HasSynced() ||
		!c.nodes.informer.HasSynced() {
		return false
	}
	return true
}
```

## Controller Loop的实现

Controller 的 Event Controller Loop 实现在 Run()方法中：

```go
func (c *Controller) Run(stop <-chan struct{}) {
    // chan stop 用于传递停止信号
	go c.queue.Run(stop)

	go c.services.informer.Run(stop)
	go c.pods.informer.Run(stop)
	go c.nodes.informer.Run(stop)

	// 为了避免没有Label或者ports的 endpoint，endpoint informer的运行需要等待nodes/pods/services同步
	cache.WaitForCacheSync(stop, c.nodes.informer.HasSynced, c.pods.informer.HasSynced,
		c.services.informer.HasSynced)

	go c.endpoints.informer.Run(stop)

	<-stop
	log.Infof("Controller terminated")
}
```

### queue 的实现

```go
type Queue interface {
	Push(Task)
	Run(<-chan struct{})
}
type queueImpl struct {
    // 延迟，发生错误时重试的间隔时间
	delay   time.Duration
    // 任务列表
	queue   []Task
    // 条件，用于线程同步
	cond    *sync.Cond
    // 是否要关闭的标志
	closing bool
}

// push方法的实现，去掉线程同步和关闭处理，其实就是两行代码
func (q *queueImpl) Push(item Task) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if !q.closing {
        // 1. 将任务存放在queue中
		q.queue = append(q.queue, item)
	}
    // 2. 发信号通知Run()进行处理
	q.cond.Signal()
}

func (q *queueImpl) Run(stop <-chan struct{}) {
    // 关闭处理
	go func() {
		<-stop
		q.cond.L.Lock()
		q.closing = true
		q.cond.L.Unlock()
	}()

	for {
		q.cond.L.Lock()
		for !q.closing && len(q.queue) == 0 {
			q.cond.Wait()
		}

		if len(q.queue) == 0 {
			q.cond.L.Unlock()
			return
		}

        // 从queue中取出一个任务
		var item Task
		item, q.queue = q.queue[0], q.queue[1:]
		q.cond.L.Unlock()

        // 执行handler
		if err := item.handler(item.obj, item.event); err != nil {
            // 如果执行失败
			log.Infof("Work item handle failed (%v), retry after delay %v", err, q.delay)
            // 延迟一段时间之后，再将这个任务放回queue，继续重试
            // 如果有任务总是返回error，岂不是要无限循环？
            // 没有看到重试次数的限制
			time.AfterFunc(q.delay, func() {
				q.Push(item)
			})
		}
	}
}
```

### Informer的实现

SharedInformer 具有共享数据缓存，并且能够将对缓存更改的通知分发给通过 AddEventHandler 注册的多个监听器。 

如果使用此方法，则与标准 Informer 相比，有一个行为有所不同。当您收到通知时，缓存将至少与通知一样新鲜，但它可能更新鲜。 您不应该依赖缓存的内容与处理函数中收到的通知完全匹配。 如果先有创建，然后是删除，则缓存可能没有您的项目。

这比广播有优势，因为它允许我们在许多控制器之间共享公共缓存。 扩展广播需要我们为每个监听保留重复的缓存。

```go
type SharedInformer interface {
    // AddEventHandler 使用共享信息器的重新同步周期(resync period)向共享信息器添加事件处理程序。 
    // 发往单个handler的事件按顺序传递，但不同handler之间没有协调。
	AddEventHandler(handler ResourceEventHandler)
    // AddEventHandler 使用特定的重新同步周期(resync period)向共享信息器添加事件处理程序。 
    // 发往单个handler的事件按顺序传递，但不同handler之间没有协调。
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore 返回存储对象.
	GetStore() Store
	// GetController 取回启动informer的合成接口
	GetController() Controller
	// Run 启动shared informer, 当stopCh 关闭时停止。
	Run(stopCh <-chan struct{})
	// HasSynced 返回true，如果shared informer的存储已经同步
	HasSynced() bool
    // LastSyncResourceVersion是上次与底层存储同步时观察到的资源版本。 
    // 返回的值与访问底层存储不同步，也不是线程安全的。
	LastSyncResourceVersion() string
}

type sharedIndexInformer struct {
	indexer    Indexer
	controller Controller

	processor             *sharedProcessor
	cacheMutationDetector CacheMutationDetector

    // 跟踪该块以处理controller的后期初始化
	listerWatcher ListerWatcher
	objectType    runtime.Object

    // resyncCheckPeriod 是我们希望reflector的resync timer触发的频率
    // reflector可以调用 shouldResync 来检查我们的监听器是否需要重新同步。
	resyncCheckPeriod time.Duration
    // defaultEventHandlerResyncPeriod 是通过 AddEventHandler添加的handler的默认重新同步周期（即，它们不指定而只想使用shared informer的默认值）。
	defaultEventHandlerResyncPeriod time.Duration
	// clock 容许进行测试
	clock clock.Clock

	started, stopped bool
	startedLock      sync.Mutex
    
    // blockDeltas 提供了一种停止所有事件分发的方法，以便后期事件处理程序可以安全地加入共享informer。
	blockDeltas sync.Mutex
}
```

关键的Run方法：

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

    // 创建Delta FIFO 作为 queue 使用
	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
    
    // 最后调用到controller.Run()
	s.controller.Run(stopCh)
}
```

继续看controller.Run()方法：

```go
// Run begins processing items, and will continue until a value is sent down stopCh.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
}
```

进入r.Run，继续看 NewReflector.Run()的实现：

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
    // wait.Until()方法每隔一段时间(r.period)就执行一次func()
	wait.Until(func() {
        // func的实现就是调用ListAndWatch()
		if err := r.ListAndWatch(stopCh); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, stopCh)
}
```

总结：Controller 通过对 client-go 中提供的 SharedIndexInformer 等机制实现了对 k8s 资源的获取（List + Watch），其调用流程大体是：

- Controller.run(): kube.Controller in pilot
- go c.services.informer.Run(stop): cache.sharedIndexInformer in k8s.io/client-go
- s.controller.Run(stopCh): cache.Controller in k8s.io/client-go
- wg.StartWithChannel(stopCh, r.Run): NewReflector.Run() in k8s.io/client-go

## Discovery的实现

Controller 实现了 ServiceDiscovery 接口，Controller 的 servicesMap 和 externalNameSvcInstanceMap 字段存储了后面需要用到的数据：

```go
type Controller struct {
	servicesMap map[model.Hostname]*model.Service
    externalNameSvcInstanceMap map[model.Hostname][]*model.ServiceInstance
}
```

Services() 方法从 Controller 的 servicesMap 中获取 service 列表：

```go
func (c *Controller) Services() ([]*model.Service, error) {
	c.RLock()
	out := make([]*model.Service, 0, len(c.servicesMap))
    // 获取所有的 service
	for _, svc := range c.servicesMap {
		out = append(out, svc)
	}
	c.RUnlock()
    // 按照 hostname 排序
	sort.Slice(out, func(i, j int) bool { return out[i].Hostname < out[j].Hostname })

	return out, nil
}
```

getService 方法就更简单了：

```go
func (c *Controller) GetService(hostname model.Hostname) (*model.Service, error) {
	c.RLock()
	defer c.RUnlock()
	return c.servicesMap[hostname], nil
}
```

InstancesByPort 方法比较复杂，删除细节处理代码，重点看主流程：

```go
func (c *Controller) InstancesByPort(hostname model.Hostname, reqSvcPort int,
	labelsList model.LabelsCollection) ([]*model.ServiceInstance, error) {
	name, namespace, err := parseHostname(hostname)
    // 根据 hostname 获取 service
	svc := c.servicesMap[hostname]
	// 检查要求查找的 port 是否在 service 的 port 列表中
	svcPortEntry, exists := svc.Ports.GetByPort(reqSvcPort)

	// 如果是 external service，直接返回
	instances := c.externalNameSvcInstanceMap[hostname]
	if instances != nil {
		return instances, nil
	}

    // 从 informer 获取到 endpoints 列表
    // 这是所有的 endpoint ？ 有点狠
	for _, item := range c.endpoints.informer.GetStore().List() {
		ep := *item.(*v1.Endpoints)
        // 根据 endpoint 的 name 和 namespace 过滤一下
        // 还好这里的过滤方式足够简单，应该速度够快
		if ep.Name == name && ep.Namespace == namespace {
			var out []*model.ServiceInstance
			for _, ss := range ep.Subsets {
				for _, ea := range ss.Addresses {
                    // 两个for循环，终于拿到ip地址
                    // 通过ip地址获取labels
					labels, _ := c.pods.labelsByIP(ea.IP)
					// 检查 labels 匹配情况
					if !labelsList.HasSubsetOf(labels) {
						continue
					}

                    // 通过ip地址获取pod
					pod := c.pods.getPodByIP(ea.IP)


					for _, port := range ss.Ports {
						if port.Name == "" || 
							reqSvcPort == 0 || 
							svcPortEntry.Name == port.Name {
                            // 生成 ServiceInstance 实例，加入out
							out = append(out, &model.ServiceInstance{
								Endpoint: model.NetworkEndpoint{
									Address:     ea.IP,
									Port:        int(port.Port),
									......
							})
						}
					}
				}
			}
			return out, nil
		}
	}
	return nil, nil
}
```

简单说，就是在 servicesMap 、externalNameSvcInstanceMap，还有 pod 、 endpoints 等存储有全量数据的情况下，根据输入条件取满足要求的数据。

