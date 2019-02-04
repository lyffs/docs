# Docker 源码分析 #

	参考地址：https://hk.saowen.com/a/df55dda06ed23b9d1a48e8f254e931ef5e7cb5bb0bc9d405cca876c804770d31


---

## 目录结构 ##

-> api
-> builder
-> cli
-> client
-> cmd # 命令行的入口
-> container # 容器的抽象
-> contrib #
-> daemon # dockerd daemon
-> distribution
-> dockerversion
-> docs
-> errdefs
-> hack
-> image #镜像的抽象
-> integration
-> integration-cli
-> internal
-> layer
-> libcontainerd
-> migrate
-> oci
-> opts
-> pkg
-> plugin
-> profiles
-> project
-> reference
-> registry
-> reports
-> restartmanager
-> runconfig
-> vendor
-> volume

	Docker的设计是单机的，不是分布式的
	Docker的设计是client-Server模式。


-> 从命令行入手

	-> func newDaemonCommand() *cobra.Command {}
		-> opts := newDaemonOptions(config.New())
		-> runDaemon(opts)
			-> daemonCli := NewDaemonCli()
			-> daemonCli.start(opts)

	-> func (cli *DaemonCli) start(opts *daemonOptions) (err error) {}

		-> serverConfig, err := newAPIServerConfig(cli)	// 设置API服务配置

		-> cli.api = apiserver.New(serverConfig) // 初始化服务配置

		-> hosts, err := loadListeners(cli, serverConfig) // 建立tcp listen,并且设置http server

		-> opts, err := cli.getContainerdDaemonOpts() // 获取ContainerdDameon 配置函数

		-> r, err := supervisor.Start(ctx, filepath.Join(cli.Config.Root, "containerd"), filepath.Join(cli.Config.ExecRoot, "containerd"), opts...) // 启动containerd daemon和监控它

			-> err := opt(r) // 执行配置函数

			-> go r.monitorDaemon(ctx) //

				-> err := r.startContainerd() // 启动containerd dameon (具体进程名是：containerd)

					-> pid, err := r.getContainerdPid() // 获取libcontainerd pid

					->  err := cmd.Start() // cmd = containerd	

					-> client, err = containerd.New(r.GRPC.Address, containerd.WithTimeout(60*time.Second)) // 返回一个连接containerd实例的客户端。

					-> _, err := client.IsServing(tctx)

		-> pluginStore := plugin.NewStore() // 新建插件商店

		-> err := cli.initMiddlewares(cli.api, serverConfig, pluginStore) // 初始化中间件

			-> good use:

			```
			// WrapHandler returns a new handler function wrapping the previous one in the request chain.
			func (e ExperimentalMiddleware) WrapHandler(handler func(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error) func(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
					return func(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
						w.Header().Set("Docker-Experimental", e.experimental)
					return handler(ctx, w, r, vars)
				}
			}
			```

		-> d, err := daemon.NewDaemon(ctx, cli.Config, pluginStore)	// 实例化daemon