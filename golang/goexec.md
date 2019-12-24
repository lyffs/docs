## Go program execute process

### 简介
	本文从Golang的小程序反汇编得到汇编代码，从中分析Golang程序执行过程。
	
	Golang 程序

	`
		package main
		import "fmt"

		func main() {
			ch1 := make(chan int)
			go func() {
				fmt.Println("hello")
				ch1 <- 1
		}()
		<-ch1
		}

	`
	先通过golang build -gcflags "-N -l" （禁止内联）生成可执行程序，然后通过GDB跟踪。

### 详情
	//入口
	1.TEXT runtime·rt0_go(SB),NOSPLIT,$0
		get_tls(BX)
	       	LEAQ    runtime·g0(SB), CX
       		MOVQ    CX, g(BX)
	        LEAQ    runtime·m0(SB), AX

		// save m->g0 = g0
	        MOVQ    CX, m_g0(AX)
       		// save m0 to g0->m
	        MOVQ    AX, g_m(CX)

		CALL    runtime·check(SB)
		MOVL    16(SP), AX              // copy argc
	        MOVL    AX, 0(SP)
       		MOVQ    24(SP), AX              // copy argv
       		MOVQ    AX, 8(SP)
       		CALL    runtime·args(SB)
	        CALL    runtime·osinit(SB)
	        CALL    runtime·schedinit(SB)

		// create a new goroutine to start program
	        MOVQ    $runtime·mainPC(SB), AX         // entry
       		PUSHQ   AX   
	        PUSHQ   $0                      // arg size
       		CALL    runtime·newproc(SB)
	        POPQ    AX   
        	POPQ    AX   

      		// start this M
		// mstart shou never return
	        CALL    runtime·mstart(SB)


	//类型测试
	2 runtime.check()

	//
	3 runtime.args()

	//
	4 runtime.osinit()
		ncpu = getproccount() //获取CPU计算
		physHugePageSize = getHugePageSize() //获取大页大小
		
	//The bootstrap sequence is:
	// call osinit
	// call schedinit
	// make & queue new G
	// call runtime.mstart

	5 runtime.schedinit()
		_g_ := getg() //获取当前g的指针
		sched.maxmcount = 10000 //m的最大计数为1w
		tracebackinit()
		moduledateverify()
		stackinit() //初始化stack pool
		mallocinit() //内存管理初始化，主要是初始花mheap_，初始化arena。定义不同内存状态的申请
		mcommoninit(_g_.m) //m公共初始化
		cpuinit() //cpu初始化
		alginit() //算法初始化
		modulesinit() //
		typelinksinit()
		itabsinit()

		msigsave(_g_.m)
		
		goargs()
		goenvs()
		parsedebuggvars() //解析debug变量
		procresize(procs) //write barrier needs a P	

	6 runtime.mallocinit()
		mheap_.init() //堆初始化
		_g_ := getg()
		_g_.m.mcache = allocmacache()

	7 runtime.mcommoninit()
		mpreinit(mp) //m.gsignal初始化，其中新建g，同时g的stack需要申请。从系统内存申请，或者从stackpool申请，或者从heap（mheap_）申请。
		
	8 runtime.procresize(nproc int32) *p
		// initialize new P's
		for i := old; i < nprocs; i++ {
			pp := allp[i]
			if pp == nil {
				pp = new(p)
			}

			pp.init(i) //设置pp的状态为:_Pgcstop,pp.mcache = getg().m.mcache //或者pp.mcache = allocmcache()
			atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
		}

	9 runtime.main()
		lockOSThread()		
		if g.m != &m0 {
			throw("runtime.main not on m0")
		} //当前工作线程必须是m0	

		doInit(&runtime_inittask) //必须要在defer之前

		gcenable() //开启gc，启动bgsweep() bgscavenge协程

		doInit(&main_inittask)
		unlockOSThread()

		fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
		fn()

		if atomic.Load(&panicking) != 0 {
			gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
		}

		exit(0)

	10 runtime.gcenable() 
		c := make(chan int, 2)
		go bgsweep(c)
		go bgscavenge(c)
		<-c
		<-c
		memstats.enablegc = true

	11 func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int)
		// 将当前goroutine设置为waiting状态，并且调用unlockf函数
		// 如果unlockf返回false，goroutine恢复
		// reason解释goroutine parked的原因
		// 它显示在堆栈跟踪和堆转储中
		
		mp := acquirem() //获取当前goroutine关联的m
		gp := mp.curg //获取m中正在运行的goroutine
		status := readgstatus(gp) //获取gp(goroutine)的状态
		mp.waitlock = lock
		mp.waitunlock = unlockf
		
		gp.waitreason = reason

		mp.waittraceev = traceEv
		mp.waittraceskip = traceskip
		
		releasem(mp)

		// func mcall(fn func(*g))
		// mcall从g切换到g0栈调用fn(g)，其中g是发起调用的goroutine。
		// mcall将g当前的PC/SP保存在g-sched以便以后恢复。
		// 由fn决定稍后的执行，通过记录g到数据结构中，从而后面可以调用ready(g)
		// 当g被重新调度时，mcall随后返回原始goroutine g。
		// fn绝对不能返回；通常通过调用schedule结束，让m来执行其他goroutine。
		// 
		// mcall只能在g stack(非g0或者gsignal)中被调用
		//
		// 函数必须是 go:noescape(不能逃逸)，如果fn是栈分配的闭包，
		// fn将g放在运行队列中，g在fn返回之前执行。闭包在执行过程中无效。

		mcall(park_m)

		
	12 runtime.newproc(siz int32, fn *funcval) 
		//创建一个新的协程用于运行fn
		//将fn放在g的队列中等待运行
		//编译器将go声明转换成对它的调用
		//协程stack不能split因为它猜测函数参数在fn后面有效排列，如果stack split，他们不会复制

		argp := add(unsafe.Pointer(&fn), sys.PtrSize) //获取参数地址
		gp := getg() //获取当前goroutine
		pc := getcallerpc()

		//systemstack: 
		systemstack(func() {
			newproc1(fn, (*uint8)(argp), siz, gp, pc)
		}) 

		
	13 runtime.park_m
		// 在g0中继续暂停
		
		_g_ := getg()
		
		//将gp状态从_Grunning变更为_Gwaiting
		casgstatus(gp, _Grunning, _Gwaiting)

		//dropg 解除m与curg的联系
		dropg()

		if fn := _g_.m.waitunlockf; fn != nil {
			ok := fn(gp, _g_.m.waitlock)
			_g_.m.waitunlockf = nil
			_g_.m.waitlock = nil
			if !ok {
				casgstatus(gp, _Gwaiting, _Grunnable)
				execute(gp, true)
			}
		}

		schedule()

	14 runtime.casgstatus(gp *g, oldval, newval uint32)
		// 如果设置为Gscanstatus或者从Gscanstatus状态变更，这样抛出异常。取而代之的是castogscanstatus 和casfrom_Gscanstatus
	        // 如果g->atomicstatus处于Gscan状态，casgstatus将会一直循环，直到设置状态为Gscan的routine完成。
		
		if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
			//throw() //在系统栈
		}	

		if oldval == _Grunning && gp.gcscanvalid {
			//throw() //在系统栈
		}

		const yieldDelay = 5 * 1000

		// 如果gp->atomicstatus处于扫描状态，循环会给GC时间完成并且将状态修改为oldval
		for i := 0; !atomic.Cas(&gp.atomicstatus, oldval, newval); i++ {
			if i == 0 {
				nextYield = nanotime() + yieldDelay
			}
			if nanotime() < nextYield {
				for x := 0; x < 10 && gp.atomicstatus != oldval; x++ {
					procyield(1)
				}
			} else {
				osyield()
				nextYield = nanotime() + yieldDelay/2
			}
		}	
		
		if newval == _Grunning {
			gp.gcscanvalid = false
		}

	15 runtime.dropg()
		// dropg将移除m和当前routine m->curg的关联
		// 通常调用者变更gp的Grunning状态然后立刻调用dropg完成该项工作
		// 调用者还负责安排gp在恰当的时间状态变更为ready重新启动。
		// 在调用dropg和稍后安排gp状态变更为ready，调用者可以做其他工作
		// 但最终应该调用schedule来重新启动这个m上面的goroutine调度。

		_g_ := getg()

		 setMNoWB(&_g_.m.curg.m, nil) //非写屏障设置m当前gouroutine的m为空
		 setGNoWB(&_g_.m.curg, nil) //非写屏障设置m当前goroutine为空

	16 runtime.execute(gp *g, inheritTime bool)
		// 调度gp在当前M上运行。
		// 如果inheritTime是true，gp在当前时间片中继承保留的时间。否则它开始一个新的
		// 时间片。
		// 从不返回

		// 允许写屏障因为在几个地方获得P之后立即调用。
		_g_ := getg()
		// 将gp的状态从_Grunnable变更为_Grunning
		casgstatus(gp, _Grunnable, _Grunning)
		gp.waitsince = 0
		gp.preemt = false
		gp.stackguard0 = gp.stack.lo + _StackGuard
		if !inheritTime {
			_g_.m.p.ptr().schedtick++
		}

		_g_.m.curg = gp //将m的正在运行的goroutine变更为gp
		gp.m = _g_.m //将gp的m变更为_g_.m

		// 检查profiler是否需要打开或者关闭
		hz := sched.profilehz
		if _g_.m.profilehz != hz {
			setThreadCPUProfiler(hz)
		}

		gogo(&gp.sched)

	17 runtime.schedule()
		// 调用执行一轮是：寻找一个可执行的goroutine并且执行。
		_g_ := getg()

		// guintptr, muintptr, and puintptr 用来绕过写屏障
		// 当 当前P被释放时，避免写屏障特别重要。因为GC认为整个
		// 程序已经停止，一个未预期的写屏障不会与GC同步，这可能导致
		// 对象已经标记但没有入队的执行到一半的写屏障。如果在入队之前
		// GC跳过该对象并且完成，这将导致不正确的释放该对象。

		// 我们尝试当没有拥有一个正在运行的P时只使用特殊的赋值函数，但是
		// 一些特殊内存单词的更新可以穿过写屏障，而有些没有。这打破写屏障阴影检查
		// 的模式，同时令人害怕的是：拥有一个完全被GC忽视的单词要比拥有一个只忽略少量
		// 更新的单词要好。

		// 在allgs和allp 列表中 或者来自栈变量（在他们到达那些列表之前的变量申请）的Gs
		// 和Ps通过正确的指针访问总是有效的。

		// 不论来自allm还是来自freem，Ms通过正确的指针访问总是有效。不像 Gs 和 Ps，我们
		// 释放 Ms，所有没有任何东西拥有越过安全点的muintptr非常重要。

		// A guintptr持有一个goroutine的指针，但是作为绕过写屏障的uintptr类型。它用在
		// Gobuf goroutine状态和不使用P操作的调度列表中。

		// Gobuf.g goroutine指针 总是被汇编代码更新。在一小部分地方中，
		// 它是由go代码更新 -func save-，它必须被看做一个uintptr为了避免在错误的时间
		// 产生写屏障。为了取代理解在非汇编操作中产生写屏障，我们改变uintptr的类型，所以
		// 它根本不需要写屏障

		// goroutine结构被发布在allg列表中并且从不释放。它将阻止goroutine结构被回收。
		// Gobuf.g 从来不是goroutine的唯一引用。在allg上发布的goroutine最新生成。
		// Goroutine的指针同时会保留在GC不可见的地方，比如：TLS，所以我不能看到它们
		// 移动过。如果我们确实想开始移动数据到GC，我们需要从恰当的arena中申请
		// goroutine结构。用guintptr指针不让错误变得更糟糕。

		if _g_.m.lockedg != 0 {
			stoplockedm()
			execute(_g_.m.lockedg.ptr(), false)
		}

	18 runtime.stoplockedm() 
		// 停止锁定到g的当前M 的执行，直到g再次执行。
		_g_ := getg()
		
		if _g_.m.p != 0 {
			// 调度其他M来运行这个P
			_p_ := releasep()
			handoffp(_p_)
		}

	19 runtime.releasep() *p
		// 断开p与当前m的联系
		_g_ := getg()
		// 获得当前m关联的p
		_p_ := _g_.m.p.ptr()
		// 当前m的p为0
		_g_.m.p = 0
		// 当前m的mcache为nil
		_g_.m.mcache = nil
		// 相应的p的m为0
		_p_.m = 0
		// 设置p的状态为空闲
		_p_.status = _Pidle
		return _p_

	20 runtime.handoffp(_p_ *p)
		// 通过系统调用或者锁定的M，挂起_p_
		// 总是不需要P运行，所以不允许写屏障。
		// handoffp 必须在findrunnable返回G并在_p_上运行的任何情况下启动M。
		
		if !runqempty(_p_) || sched.runqsize != 0 {
						
		}

	21 runtime.runqempty(_p_ *p) bool 
		// runqempty报告_p_在本地运行队列中是否有Gs。
		// 假设它从不返回true
		
		for {
			head := atomic.Load(&_p_.runqhead)
			tail := atomic.Load(&_p_.runqtail)
			runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&_p_.runnext)))
			// p.runnext，如果非空，是一个由当前G准备的可运行的G。
			// 如果正在运行的G还有时间片，那么接着运行的应该是runnext，而不是
			// 在runq中Gs。它将继承当前时间片剩余的时间。
			// 如果一部分的goroutine被锁定为通信与等待的模式，被设置为一个单元的调度
			// 消除由于将处于准备状态的goroutine添加到队列末尾而产生（可能）很大
			// 调度延迟。

			if tail == atomic.load(&_p_.runqtail) {
				return head == tail && runnext == 0
			}
		}

	22 runtime.startm(_p_ *p, spinning bool)
		// 调度一些M来运行P(如果需要会创建新的M)
		// 如果p为空，尝试获取一个空闲的p，如果没有空闲p怎不做任何处理。
		// 可能运行时m.p为空，所以不允许写屏障。
		// 如果设置了spinning，调用者已经增加nmspinning和startm将减少nmspinning或者在新启动的M中设置m.spinning
		
		lock(&sched.lock)
		if _p_  == nil {
			// 尝试从空闲列表中返回p
			// sched必须被锁定
			// 可能y运行中处于STW，所以不允许写屏障。
			_p_  = pidleget() 
			if _p_ == nil {
				unlock(&sched.lock)
				if spinning {
					
				}
				return
			}
		}

		mp := mget()
		// 尝试从空闲链表中返回m
		unlock(&sched.lock)

		if mp == nil {
			var fn func()
			if spinning {
				fn = mspinning
			}
			newm(fn, _p_)
			return
		}

	23 runtime.newm(fn func(), _p_ *p)	
		// 创建一个新的m，它开始于fn或者调度器的调用
		// fn 需要是静态的并且不是堆申请的闭包
		// 运行的时候m.p可能为nil，所以不允许写屏障

		mp := allocm(_p_, fn)


	24 runtime.allocm(_p_ *p, fn func()) *m
		// 申请一个和任何线程无关的m
		// 如果需要的话，可以把p用作申请的上下文。
		// fn被记录为新m的m.mstartfn

		// 该函数运行写屏障即使调用者没有这样做，因为它借用了_p_
		//  
		_g_ := getg() //获取当前goroutine
		acquirem() // 禁止GC因为在sysmon中被调用
		if _g_.m.p == 0 {
			// 在函数中为了mallocs借用p
			// 把p和当前m关联起来
			// 
			acquirep(_p_)
		}

	25 runtime.acquirep(_p_ *p)
		// 这部分的执行不允许写屏障
		// wirep是acquirep的第一步，它实际上是将当前M和_p_关联起来。
		// 所以这部分我们不允许写屏障，因为我们没有一个P。
		// 设置_g_.m.p.set(_p_)
		// _p_.m.set(_g_.m)
		wirep(_p_)	
		// 拥有P，现在允许写屏障

		// 在P从可能旧的mcache中申请之前，执行延迟的mcache刷新
		_p_.mcache.prepareForSweep()

	26 runtime (c *mcache)prepareForSweep()
		// 当c已经发布，prepareForSweep会刷新c如果系统已经进入一个新的sweep(扫描)阶段
		// 这肯定发生在sweep阶段开始和第一次申请之间。

		// 另外，为了替换 我们确保每一个P在(starting the world)和分配之间的都这样做，
		// 我们可以开启allocte-black，允许申请像平常一样继续，用一个ragged barrier在
		// 扫描的开始来确保所有缓存的spans被扫描，然后禁用allocate-black。然而，出于
		// 这个目的很难避免位标记蔓延至下个GC循环。

		sg := mheap_.sweepgen
		if c.flushGen == sg {
			return
		} else if c.flushGen != sg-2 {
			throw("bad flush")
		}

		c.releaseAll()

	27 runtime (c *mcache)releaseAll
		for i := range c.alloc {
			s := c.alloc[i]
			if s != &emptymspan {
				// s非空mspan
				mheap_.central[i].mcentral.uncacheSpan(s)
				c.alloc[i] = &emptymspan
			}
		}
		
	28 runtime (c *mcentral) uncacheSpan(s *mspan)
		if s.allocCount == 0 {
			throw("uncaching span but s.allocCount == 0")
		}

		sg := mheap_.sweepgen
		// 
		stale := s.sweepgen == sg+1
		if stale {

		} else{

		}

	29 runtime.mspan
		//go:notinheap
		
		// 链表中下一个mspan，如果没有为nil
		next *mspan
		// 链表中前一个mspan，如果没有为nil
		prev *mspan
		// 用作debug，TODO: Remove
		list *mSpanList

		// span第一个字节的地址，又名s.base()
		startAddr uintptr
		// span中页数目
		npages uintptr 	

		// 在mSpanManual mspan中的空闲对象链表
		manualFreeList gclinkptr

		// 
		freeindex uintptr
	

	30 runtime.newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr)
		// 创建一个运行fn从argp开始拥有narg字节参数的协程。callerpc是创建它的go语句地址。
		// 新的g被放到等待运行的g队列中。
		_g_ := getg() //获取当前goroutine
		acquirem() // 

		_p_ := _g_.m.p.ptr() //返回PP指针
		newg := gfget(_p_)

	31 runtime.gfget(_p_ *p) *g
		// 从g空闲列表中获取
		// 如果本地列表是空，则从全局列表中获取一批

		retry:
		if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
			lock(&sched.gFree.lock)
			//将一批空闲的Gs移到P
			if _p_.gFree.n < 32 {
				//优先获取有栈的Gs，同时如果 sched.gFree.stack.head = gp.schedlink
				gp := sched.gFree.stack.pop()
				if gp == nil {
					//
					gp = sched.gFree.noStack.pop()	
					if gp == nil {
						break
					}
				}
				sched.gFree.n--
				_p_.gFree.push(gp) //以入栈的方式将goroutine 连接起来
				_p_.gFree.n++ 
			}
			unlock(&sched.gFree.lock)
			goto retry
		}

		gp := _p_.gFree.pop() //弹出最后进栈的goroutine
		if gp == nil {
			return nil
		}

		_p_.gFree.n--
		if gp.stack.lo == 0 {
			// stack 已经在 gfput 中回收，重新申请一个新的
			systemstack(func() {
				gp.stack = stackalloc(_fixedStack)	
			})
			gp.stackguard0 = gp.stack.lo + _StackGurad	
		} else {
			if raceenabled {
				racemalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
			}
			if msanenabled {
				msanmalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
			}	
		}

		return gp

	32 runtime.stackalloc(n uint32) stack
		// 申请b个自己的栈
		// stackalloc必须运行在系统栈上面因为它用每个P的资源，并且不能缩放堆栈。
		// Stackalloc必须运行在scheduler栈，因为我们从不尝试在stackalloc运行的代码期间
		// 增加堆栈 


### 参考
	https://docs.oracle.com/cd/E19205-01/820-1200/blaoy/index.html

