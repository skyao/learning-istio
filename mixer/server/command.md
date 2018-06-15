# Mixer Server命令

和命令行命令相关的代码在 `istio/mixer/cmd/mixs/cmd` 下。

### GetRootCmd()函数

`istio/mixer/cmd/mixs/cmd/root.go` 文件中的 `GetRootCmd()` 函数返回 cobra command-tree 的根。

```go
func GetRootCmd(args []string, info map[string]template.Info, adapters []adapter.InfoFn,
	printf, fatalf shared.FormatFn) *cobra.Command {
    rootCmd := &cobra.Command{......},
	}
	rootCmd.SetArgs(args)
	rootCmd.PersistentFlags().AddGoFlagSet(flag.CommandLine)

	// 添加服务器命令
	rootCmd.AddCommand(serverCmd(info, adapters, printf, fatalf))
	rootCmd.AddCommand(crdCmd(info, adapters, printf, fatalf))
	rootCmd.AddCommand(probeCmd(printf, fatalf))
	rootCmd.AddCommand(version.CobraCommand())
	rootCmd.AddCommand(collateral.CobraCommand(rootCmd, &doc.GenManHeader{
		Title:   "Istio Mixer Server",
		Section: "mixs CLI",
		Manual:  "Istio Mixer Server",
	}))

	return rootCmd
}
```



### serverCmd()函数

`istio/mixer/cmd/mixs/cmd/server.go` 文件中的 `serverCmd()` 设置服务器命令：

```go
func serverCmd(info map[string]template.Info, adapters []adapter.InfoFn, printf, fatalf shared.FormatFn) *cobra.Command {
	sa := server.DefaultArgs()
	sa.Templates = info
	sa.Adapters = adapters

	serverCmd := &cobra.Command{
		Use:   "server",
		Short: "Starts Mixer as a server",
		Run: func(cmd *cobra.Command, args []string) {
			runServer(sa, printf, fatalf)
		},
	}
    
    // 设置命令行参数
    serverCmd.PersistentFlags().Uint16VarP(&sa.APIPort, "port", "p", sa.APIPort,
        "TCP port to use for Mixer's gRPC API, if the address option is not specified")
    ......
    
}
```









