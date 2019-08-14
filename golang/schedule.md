# 调度器 # 

参考：https://zhuanlan.zhihu.com/p/45959147
https://povilasv.me/#

-> 基本概念：
	G: goroutine, 每个G代表一个goroutine
	M: 工作线程, 是GO语言定义出来在用户层面描述系统线程的对象, 每个M代表一个系统线程。
	P: 处理器, 它包含了运行GO代码的资源。

	3者简要的关系是P拥有G, M必须要和一个P关联才能运行P拥有的G。

-> 节本概述:
	在Go中，线程是运行goroutine的实体，调度器的功能就是把可运行的goroutine分配到工作线程上。

	调度器4个部分：
	1.全局队列
	2.P的存放G本地队列
	3.P列表 （个数=GOMAXPROCS）
	4.M

	Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表一个内核线程，OS调度器负责把内核线程
	分配到CPU的核上执行。

-> 生命周期



==

	G = goroutine, P = logical processors  M = OS thread (execute G)

	跟踪Schedule(调度器)， 设置环境变量： GODEBUG=scheddetail=1,schedtrace=1000

	简单：GODEBUG=schedtrace=1000 ./msgserv.exe


