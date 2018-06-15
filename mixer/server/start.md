# 服务器启动

## 启动入口

Mixer服务器的启动入口在`istio/mixer/cmd/mixs/main.go` 中的main函数：

```go
func main() {
    // 创建 cobra.Command 对象
	rootCmd := cmd.GetRootCmd(os.Args[1:], supportedTemplates(), supportedAdapters(), shared.Printf, shared.Fatalf)

    // 执行server命令
	if err := rootCmd.Execute(); err != nil {
		os.Exit(-1)
	}
}
```

supportedTemplates() 方法和 supportedAdapters() 方法用来获取支持的template和adapter列表：

```go
func supportedTemplates() map[string]template.Info {
	return generatedTmplRepo.SupportedTmplInfo
}

func supportedAdapters() []adptr.InfoFn {
	return adapter.Inventory()
}
```

server命令和cobra.Command的使用具体见章节 [server命令](command.md)。

## 执行server命令

在GetRootCmd()中会调用serverCmd()方法，添加server命令：

> 备注：GetRootCmd()的代码在文件 `istio/mixer/cmd/mixs/cmd/root.go` 中

```go
rootCmd.AddCommand(serverCmd(info, adapters, printf, fatalf))
```

 `mixs server` 命令的定义如下：

> 备注：serverCmd()的代码在文件 `istio/mixer/cmd/mixs/cmd/server.go` 中

```go
func serverCmd(info map[string]template.Info, adapters []adapter.InfoFn, printf, fatalf shared.FormatFn) *cobra.Command {
	......
	serverCmd := &cobra.Command{
		Use:   "server",
		Short: "Starts Mixer as a server",
		Run: func(cmd *cobra.Command, args []string) {
			runServer(sa, printf, fatalf)
		},
	}
    ......
}
```

在 server 命令执行时，会调用runServer()方法，创建一个server然后运行:

```go
func runServer(sa *server.Args, printf, fatalf shared.FormatFn) {
	......
	s, err := server.New(sa)
	s.Run()
	......
}
```

New 方法初始化一个全功能的Mixer server, 准备接收流量：

> 备注：New()的代码在文件 `istio/mixer/cmd/mixs/server/server.go` 中

```go
func New(a *Args) (*Server, error) {
	return newServer(a, newPatchTable())
}
```

## 创建服务器

newServer()是最重要的代码了，这里创建server：

```go
func newServer(a *Args, p *patchTable) (*Server, error) {
    // 创建server结构体，设置各种参数
    s := &Server{}
	s.gp = pool.NewGoroutinePool(apiPoolSize, a.SingleThreaded)
	s.gp.AddWorkers(apiPoolSize)
    
    s.adapterGP = pool.NewGoroutinePool(adapterPoolSize, a.SingleThreaded)
	s.adapterGP.AddWorkers(adapterPoolSize)
    
    // 准备template和adapter
    tmplRepo := template.NewRepository(a.Templates)
	adapterMap := config.AdapterInfoMap(a.Adapters, tmplRepo.SupportsTemplate)
    ......
    // 创建runtime
    rt = p.newRuntime(st, templateMap, adapterMap, a.ConfigIdentityAttribute, a.ConfigDefaultNamespace, s.gp, s.adapterGP, a.TracingOptions.TracingEnabled())

	if err = p.runtimeListen(rt); err != nil {
		_ = s.Close()
		return nil, fmt.Errorf("unable to listen: %v", err)
	}
	s.dispatcher = rt.Dispatcher()
    
    // 创建grpc server
    s.server = grpc.NewServer(grpcOptions...)
    // 注册mixer service到grpc server
	mixerpb.RegisterMixerServer(s.server, api.NewGRPCServer(s.dispatcher, s.gp))
    
    // 启动
    go ctrlz.Run(a.IntrospectionOptions, nil)

	return s, nil
}
```

## 注册Mixer Service

newServer()中的这行代码，在启动的gRPC server中，注册了mixer service:

> 备注：RegisterMixerServer()的代码在文件 `istio/vendor/istio.io/api/mixer/v1/service.pb.go` 中

```go
mixerpb.RegisterMixerServer(s.server, api.NewGRPCServer(s.dispatcher, s.gp))
```

其中RegisterMixerServer()方法的代码如下，通过调用grpc.Server()方法将gRPC服务定义和gRPC服务实现注册到gRPC server：

```go
func RegisterMixerServer(s *grpc.Server, srv MixerServer) {
	s.RegisterService(&_Mixer_serviceDesc, srv)
}
```

这里的 `_Mixer_serviceDesc` 来自文件 `istio/vendor/istio.io/api/mixer/v1/service.pb.go` ，这是 mixer api 中定义的 mixer service，其 protocolbuf 文件内容为：

```protobuf
service Mixer {
  rpc Check(CheckRequest) returns (CheckResponse) {}
  rpc Report(ReportRequest) returns (ReportResponse) {}
}
```

这个Mixer service 就是 mixer 和 envoy 之间的通讯接口，envoy 通过调用 check() 和 report() 方法来和mixer进行通讯。

而 `api.NewGRPCServer(s.dispatcher, s.gp)` 负责创建 mixer service 的实例，用于后续的注册(到grpc server)：

```go
func NewGRPCServer(dispatcher dispatcher.Dispatcher, gp *pool.GoroutinePool) mixerpb.MixerServer {
	......
	return &grpcServer{
		dispatcher:     dispatcher,
		gp:             gp,
		globalWordList: list,
		globalDict:     globalDict,
	}
}
```

这里的 grpcServer 就是 mixer service 的实现。具体代码和流程见后面的章节 [check主流程](check.md) 。

## 运行服务器

Run()方法运行服务器，让mixer开始在主API端口接收grpc请求：

```go
func (s *Server) Run() {
	s.shutdown = make(chan error, 1)
	s.SetAvailable(nil)
	go func() {
		// go to work...
		err := s.server.Serve(s.listener)

		// notify closer we're done
		s.shutdown <- err
	}()
}
```

TBD: server.Serve()里面有一堆细节，以后再细读。

