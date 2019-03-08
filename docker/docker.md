# Docker 源码分析 #

	参考地址：https://www.kancloud.cn/infoq/docker-source-code-analysis/80525


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

			-> good code:

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

			-> setDefaultMtu(config) // 设置mtu=1500

			-> registryService, err := registry.NewService(config.ServiceOptions) // 新建registry service

			-> err := ModifyRootKeyLimit() // 确保 root_key_limit 正确,判断内核值：kernel.keys.root_maxkeys

			-> err := verifyDaemonSettings(config) // 检查配置是否正确

			-> setupResolvConf(config)  // 设置resolve.conf /run/systemd/resolve/resolv.conf

			-> err := checkSystem() // docker daemon 运行需要root权限，kernel版本最低为 3.10.0

			-> idMapping, err := setupRemappedRoot(config) // 据配置映射容器用户和组的

			-> rootIDs := idMapping.RootPair() // 

			-> err := setupDaemonProcess(config) // 设置daemon 进程

				-> setupOOMScoreAdj(score int) // 修改OOM 阈值，/proc/self/oom_score_adj

				-> err := setMayDetachMounts() // /proc/sys/fs/may_detach_mounts

			-> 	tmp, err := prepareTempDir(config.Root, rootIDs) // 创建tmp文件夹

			-> d := &Daemon{
					configStore: config,
					PluginStore: pluginStore,
					startupDone: make(chan struct{}),
				}

			-> d.setupDumpStackTrap(stackDumpDir)
			
				-> path, err := stackdump.DumpStacks(root)	// 将goroutine stack 堆栈信息写文件

			-> err := d.setupSeccompProfile(); // 设置SeccompProfile

			-> configureMaxThreads(config) // 文件/proc/sys/kernel/threads-max值的 90%

			-> ensureDefaultAppArmorProfile() // AppArmor linux安全应用，控制应用的各种权限

			-> idtools.MkdirAllAndChown(daemonRepo, 0700, rootIDs);

			-> metricsSockPath, err := d.listenMetricsSock() //

			-> c, err := createAndStartCluster(cli, d) // 创建和启动swarm集群

			-> d.RestartSwarmContainers()

				-> err := daemon.containerStart(c, "", "", true)

					-> err := daemon.conditionalMountOnStart(container)

					-> err := daemon.initializeNetworking(container)

						-> 	err := daemon.allocateNetwork(container) // 申请网络

							-> err := controller.SandboxDestroy(container.ID)

								-> func (sb *sandbox) Delete() error

									-> c.getNetworkFromStore(ep.getNetwork().ID())

									-> ep.Leave(sb)

										-> sb.joinLeaveStart()

										-> ep.sbLeave(sb, false, options...) // endpoint leave sandbox

											-> n, err := ep.getNetworkFromStore()

											-> ep, err = n.getEndpointFromStore(ep.ID())

											-> extEp := sb.getGatewayEndpoint()

											-> d.RevokeExternalConnectivity(n.id, ep.id)

												-> network, err := d.getNetwork(nid) //d: driver, 根据nid(网络id)获取网络

												-> endpoint, err := network.getEndpoint(eid) //根据eid(endpoint id)获取endpoint

												-> err = network.releasePorts(endpoint) //释放endpoint端口

													->	err := n.releasePort(m)

														-> host, err := bnd.HostAddr()

														-> n.portMapper.Unmap(host) // bridgenetwork unmap （主机:端口）

															-> 	

											-> d.Leave(n.id, ep.id)

										-> sb.joinLeaveEnd()


 
								-> DeleteConntrackEntries(nlh *netlink.Handle, ipv4List []net.IP, ipv6List []net.IP)

									-> flowPurged, err := purgeConntrackState(nlh, syscall.AF_INET, ipAddress)

									-> 

		

												