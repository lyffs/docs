## Goroutine

### 简介
	在基于线程的基础上，在用户态设计用户态线程模型与调度器，使得调度不需要陷入内核完成，减少并发调用的代价。go在runtime层实现的是M:N的用户态线程+调度器，达到充分利用多核+更少的上下文切换代价。GM模型 -> GPM模型

### 详解
#### GM模型
	调度器是 M:1 线程池+任务队列的形式。每条内核级线程运行一个M，每个M不停的从任务队列shed中取出任务G并执行。该设计存在的问题：https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#!

#### GPM模型
	通过work stealing算法实现了M:N的G-P-M模型，同时引入逻辑Processer P。	
	基本步骤：
	1.P优先从本地的G队列中获取任务执行。
	2.G存在P队列中。当某个M陷入系统调用时，P将于M解绑定，并找另外的M（或者创建新M）来执行G代码，避免G在M之间传递。
	3.P给M运行提供了运行所需的mCache内存，当PM解绑时间回收内存。

	G代表goroutine，创建后即被放在P本地队列或者全局队列中，等待被执行。stack字段保存了goroutine栈的可用空间。在sched字段保存了goroutine切换所需的全部上下文。

	P代表processer，数目为GOMAXPROCS，通常设置为CPU核数。P不仅维护了本地的G队列，同时为M运行提供内存资源mCache，当M陷入阻塞的系统调用导致P和M分离时，P可以回收内存资源。

	M表示machine，其中mstartfn是M线程创建时的入口函数地址。当M与某个P绑定attached后mcache域将获得P的mCache内存，同时获得P和当前G。然后M进行schedule循环。shedule的机制是：
	1.从当前P的队列、全局队列或者别的P队列中findrunableG
	2.栈指针从调度器栈m->go切换到G自带的栈内存，并在此空间分配栈帧，执行G函数。
	3.执行完后调用goexit做清理工作并回到M，如此反复。

	如果遇到G切换，要将上下文都保存G当中，使得G可以被任何P绑定的M执行。M只是根据G提供的状态和信息做相同的事情。

#### Go调度器解决的问题
##### 阻塞问题
	如果任务G陷入阻塞的系统调用中，内核线程M将一起阻塞。
	封装系统调用entersyscallblock，使得进去阻塞的系统调用前执行releaseP和handoffP，即剥离M拥有的P的mCache。如果P本地队列还有G，P将去找别的M或者创建新的M来执行，若没有则直接放到pidle全局链表。当有新的G加入时可以通过WakeP获取这个空闲P。
	另外，进入非阻塞的系统调用entersyscall时会将P设置为Psyscall。监控线程sysmon在遍历P，发现Psyscall时执行handoffP。

##### 抢占调度
	go没有实现像内核一样的时间分片，设置优先级等抢占调度，只引入了初级的抢占。监控线程sysmon会循环retake，发现阻塞与系统调用或运行了较长时间（10ms）就会发起抢占。		
