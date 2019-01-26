# Docker 源码分析 #

	参考地址：https://hk.saowen.com/a/df55dda06ed23b9d1a48e8f254e931ef5e7cb5bb0bc9d405cca876c804770d31


---

## 目录结构 ##

..
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
