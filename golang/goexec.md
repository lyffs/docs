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

		// 释放空闲M列表。我们在某些地方做这个工作同时这可能释放我们可以使用的一个栈。
		if sched.freem != nil {
			lock(&sched.lock)
			for freem := sched.freem; freem != nil; {
				if freem.freewait != 0 {
					next := freem.freelink
					freem.freelink = newList
					newList = freem
					freem = next
					continue
				}
				stackfree(freem.g0.stack)
				freem = freem.freelink
			}
			sched.freem = newList
			unlock(&sched.lock)
		}
		
		mp := new(m)
		mp.mstartfn = fn
		mcommoninit(mp)

		

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
		// 从mcache中返回span
		if s.allocCount == 0 {
			throw("uncaching span but s.allocCount == 0")
		}

		sg := mheap_.sweepgen
		// 
		stale := s.sweepgen == sg+1
		if stale {
			// 在标记开始之前，span已经缓存，扫描它是我们的责任。
			// 设置sweepgen来指明它没有缓存但是需要标记，但是不能从
			// 这申请。sweep将设置s.sweepgen来指明s正在扫描
			atomic.Store(&s.sweepgen, sg-1)
		} else{
			// 指明s不再需要标记 
			atomic.Store(&s.sweepgen, sg)
		}

		n := int(s.nelems) - int(s.allocCount)
		if n > 0 {
			// cacheSpan更新alloc，猜想s上面的所有对象都被申请。调整为任何没有
			// 在潜在的标记span之前 我们必须做这个。
			atomic.Xadd64(&c.nmalloc, -int64(n))
			lock(&c.lock)
			c.empty.remove(s)
			c.nonempty.insert(s)
			if !stale {
				// mCentral_CacheSpan 保守的计算heap_live中未分配的槽位，撤销这个
				// 如果标记前span已经缓存，接着heap_live完全是重新计算的，因为缓存了缓存了这个span，所以我们对于陈旧的
				// span作这样的操作。
				// 
				atomic.Xadd64(&memstat.heap_live, -int64(n)*int64(s.elemsize))
			}
			unlock(&c.lock)
		}

		if stale {
			// 现在s正在正确的mcentral列表中，我们可以标记它。
			s.sweep(false)
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

		// freeindex 是从0到nelems之间的索引 在span中用来扫描下一个空闲的对象
		// 每次申请从freeindex开始扫描allocBits直到遇到0，用来指示一个空闲的对象
		// freeindex随后调整所以下次扫描开始于刚刚发现的空闲对象
		
		// 如果freeindex == nelem，这span没有空闲的对象
		
		// allocBits是span中对象的位图，如果n是大于freeindex并且allocBits[n/8] & (1<<(n%8)) == 0
		// 该对象n是空闲的；否则，对象n已经被申请。
		// 开始于nelem的Bits还没有被定义，所以不能被引用。
		// 
		// 对象n开始于地址 n*elemsize + (start << pageShift)
		freeindex uintptr

		// span中对象的数目
		nelems uintptr
		
		// 在freeindex中缓存allocBits，移动allocCache，使得最低位对应index的位。
		// allocCache 持有allocBits的补充，因此允许ctz(count trailing zero)直接使用它。
		// allocCache可能包含超越s.nelems的位；调用者必须忽视它。
		allocCache uint64

		// allocBits和gcmarkBits持有指向span标记和申请位的指针。这些指针是8字节对齐。
		// 由三块持有数据的区域
		// free: 不再访问的Dirty区域可以被重用
		// next: 持有下个GC循环使用到信息
		// current: 当前GC循环中使用的信息
		// previous: 上一个GC循环中使用的信息
		// 一个新的GC循环开始于finishsweep_m的调用。finishsweep_m将previous区域移到free
		// 区域，current区域移到previous区域，next区域移到current区域，
		// next区域将被填充，因为span请求内存为下一个GC循环保存gcmarkBits，并为新分配的
		// span保存allocBits
		
		// 指针运算是靠手工完成，取代数组为了避免在关键路径上进行边界检查。
		// 标记阶段将会释放旧的allocBits，同时设置allocBits为gcmarkBits。gcmarkBits
		// 被替换为一个新的零化内存。
		allocBits *gcBits
		gcmarkBits *gcBits

		// 标记代数
		// 当 sweepgen == h->sweepgen-2, span需要被标记
		// 当 sweepgen == h->sweepgen-1, span正在被标记
		// 当 sweepgen == h->sweepgen, span已经被标记和准备使用
		// 当 sweepgen == h->sweepgen+1, 在开始扫描之前缓存了span，现在仍然缓存，需要进行扫描
		// 当 sweepgen == h->sweepgen+3, span被扫描，接着缓存，仍然需要缓存
		// 每次GC之后，h->sweepgen增加2

		sweepgen uint32
		// 
		divMul uint16
		// 如果非零，elemsize是2的幂，&它会得到对象分配基地址
		baseMask uint16
		// 已经分配的对象数目
		allocCount uint16
		// 类型级别和nosacn
		spanclass spanClass	

		// mspan是基于页面的
		// 当msapn处于heap free treap(空闲树堆)，状态为mSpanFree, 同时heapmap(s->start)等于span;
		// heapmap(s->stars+s->npages-1) == span。
		// 当mspan处于heap scav treap(scav树堆), 那么除了上述scavenged=true，在其他情况下，scavenged=false。
		//
		// 当一个mspan被申请时，状态等于mSpanInUse或者mSpanManual，heapmap(i)等于span 对于所有的s->start <= i < s->start+s->npages
		// 每个mspan都是在双向链表中，不是在mheap的busy链表中，就在其中一个mcentral的span链表中。
		
		// mspan代表着真实内存，拥有这些状态：mSpanInUser，mSpanManual或者mSpanFree。这些状态的变迁限制如下：
		// * span可能从空闲跃迁为in-use或者manual在任何GC阶段
		// * 在标记过程(gcpase == _GCoff)，span可能从in-use跃迁为free(作为标记的结果)，或者manual跃迁为free(作为栈被释放的结果)
		// * 在GC(gcphase != _GCoff), span 不能从in-use或者manual跃迁为free。因为并发GC可能读一个指针，然后查找它的span，span
		// 状态必须是单调不变的。
		// mSpanInUse 支持垃圾回收的堆申请
		// mSpanManual 手动管理的申请
		state mSpanState
		
		// 在申请之前是否需要零化
		needzero uint8
		// 除以elemsize - divMagic.shift
		divShift uint8
		// 除以elemsize - divMagic.shift2
		divShift2 uint8
		// span的页是否释放给回OS
		scavenged bool
		// sizeclass或者npages计算得到
		elemsize uintptr
		// span的结束数据
		limit uintptr
		// 守卫特殊队列
		speciallock mutex
		// 按偏移量排序的特殊记录的链表
		// specials *special

	30 runtime.stackcache_clear(c *mcache)
		lock(&stackpoolmu)	
		for order := uint8(0); order < _NumStackOrders; order++ {
			x := c.stackcache[order].list
			for x.ptr() != nil {
				y := x.ptr().next
				stackpoolfree(x, order)
				x = y
			}
			c.stackcache[order].list = 0
			c.stackcache[order].size = 0
		}
		unlock(&stackpoolmu)
	
	31 runtime.stackpoolfree(x gclinkptr, order uint8)
		// 将栈x加入到空闲pool，调用时必须持有stackpoolmu
		s := spanOfUnchecked(uintptr(x))
		if s.state != mSpanManual {
			// msapn申请区域不是stack，throw
			throw("")
		}
		if s.manualFreeList.ptr() == nil {
			stackpool[order].insert(s)
		}
		x.ptr().next = s.manualFreeList
		s.manualFreeList = x
		s.allocCount--
		
		if gcphase == _GCoff && s.allocCount == 0 {
			// Span是完全空闲，如果我们正在标记的话，立刻返回给heap。
			// 如果GC是活跃，我们推迟释放直到GC结尾，为了避免如下类型的场景
			// 1) GC开始，扫描一个SudoG但是还没有标记SudoG.elem指针
			// 2) 指向栈的指针被复制
			// 3) 旧的栈被释放
			// 4) 包含的span被标记为空闲
			// 5) GC尝试标记SudoG.elem指针。标记失败因为指针看起来在空闲span里面。
			// 
			stackpool[order].remove(s)
			s.manualFreeList = 0
			osStackFree(s)
			mheap_.freeManual(s, &memstat.stacks_inuse)
		}
		

	32 runtime.spanOfUnchecked(p uintptr) *mspan 
		// 先通过指针p找出p所属的arena index，然后再通过mheap_.arenas的映射关系找到响应的mspan
		//go:nosplit
		// spanOfUnchecked 等价于 spanOf，不过调用者必须确保指针p处于可申请的堆区域中。
		ai := arenaIndex(p)
		return mheap_.arens[ai.l1()][ai.l2()].spans[(p/pageSize)%pagesPerArena]

	33 runtime.spanOf(p uintptr) *mspan
		// spanOf返回p所属的span。如果p不是指向heap arena或者没有span包含指针p，spanOf返回nil
		// 如果p不是指向已经申请的内存，有可能返回不包含p的non-nil的span。如果可能，调用者应该
		// 调用spanOfHeap或者明确地检查span边界。
		// 必须是nosplit因为调用者是nosplit

		// 这个函数看起来大，但是我们在arenaL1Bits上面使用大量的常量用来将它控制在预算之内。另外
		// 这里很多检查都是Go不论如何都是需要作的安全检查，所以生产代码非常短。
		ri := arenaIndex(p)
		if arenaL1Bits == 0 {
			if ri.l2() >= uint(len(meahp_.arenas[0])) {
				return nil
			}
		} else {
			if ri.l1() >= uint(len(mheap_.arenas)) {
				return nil
			}
		}
		
		l2 := mheap_.arenas[ri.l1()]
		if arenaL1Bits != 0 && l2 == nil {
			return nil
		}

		ha := l2[ri.l2()]
		if ha == nil {
			return nil
		}

		return ha.spans[(p/pagesize)%pagesPerArena]
	

	33 runtime.mheap
		//go:noinheap
		
		// heap本身就是"free"和"scav"树堆，但是其他的全局数据也在这里。
		// mheap不能是堆分配的因为它包含了不能堆分配的mSpanLists。

		// 锁只能在系统栈获取，否则g可能死锁如果栈扩展的时候持有锁。
		lock mutex
		// 空闲span
		free mTreap
		// 标记generation
		sweepgen uint32
		// 所有span被标记
		sweepdone uint32
		// 活跃sweepone调用次数
		sweepers uint32
		
		// allspans是所有创建过的mspans切片。每个mspan只出现一次。
		// allspans的内存手工管理，可以重新申请和随着heap增长移动
		// 
		// 通常，allspans由mheap_.lock保护，阻止并发访问和备份存储的释放
		// 在STW期间访问可能没有持有锁，但必须确保在访问期间不能发生申请
		allspans []*mspan

		// sweepSpans维护两个mspan栈，一个是标记在使用的mspan，另外一个
		// 未标记在使用的msapn。在每次GC期间有2种主要的角色。由于在每次GC循环中，
		// sweepgen增加2，这就意味标记的spans在sweepSpans[sweep/2%2]和未标记的
		// spans在sweepSpans[1-sweepgen/2%2]。标记从未标记的stack pop出来的span,
		// push 仍然in-use的spans入栈。同样地，分配一个正在使用的span会push到
		// 以标记的栈
		sweepSpans [2]gcSweepBuf

		// 这些参数表示从heap_live到页面扫描计数的一个线性函数。均衡的sweep
		// 系统给标记为黑色在当前heap_live通过将当前扫描计数保持在该线上面


		// 这条线的斜率为sweepPagesPerByte，通过一个基点(sweepHeapLiveBasis，pagesSweptBasis)
		// 在任何时间，系统位于(memstats.heap_live，pagesSwept)在这片空间。
		// 重要的是，这条直线穿过我们控制的一个点，而不是简单地从原点(0,0)开始，因为这样
		// 可以让我们在考虑当前进程的同时调整扫描速率。如果我们只是调整斜率，它会产生不连续
		// 债务如果有任何进展的话。

		// 统计中的spans页
		pagesInUse uint64
		// 这个循环中标记页面，原子更新
		pagesSwept uint64
		// pagesSwept 用来作为原始标记比率，原子更新
		pagesSweptBasis uint64
		// heap_live中的值用来作为原始标记比率
		sweepHeapLiveBasis uint64
		// 均衡的标记比率
		sweepPagesPerByte float64

		
		// 回收速率参数
		// 2个基础参数和回收比率平衡均衡的标记实现，基本区别是:
		// *回收关注RSS，估计为heapRetained()
		// *不是推送回收到GC，它被定步到一个基于时间的速率计算在gcPaceScavenger

		// scavengeRetainedGoal代表这我们的目标RSS
		// 所有的字段访问必须通过锁
		// 
		scavengeTimeBasis int64
		scavengeRetainedBasis uint64
		scavengeBytesPerNS float64
		scavengeRetainedGoal uint64
		scavengeGen uint64

		// 页回收状态
		// recaimIndex是在下一页allArens的页面索引用来回收，特别是
		// 它指向arena allArenas[i / pagePreArena]的page(i%pagesPerArena)
		// 如果它大于等于 1<<63，页面reclaimer已经做了扫描页面标记
		reclaimIndex uint64

		// reclaimCredit是用于额外页面扫描的备用信贷。由于页面reclaimer工作
		// 在大量的块中，他可能相比于请求回收更多。任何备用页面释放会回到
		// 信贷pool
		reclaimCredit uintptr
		
		// Malloc统计
		// 用于大对象申请的字节数
		largealloc uint64
		// 大对象申请的数目
		nlargealloc uint64
		// 用于大对象的空闲字节数 >maxsmallsize
		largefree uint64
		// 空闲大对象数目
		nlargefree uint64
		// 空闲小对象数目
		nsmallfree [_NumSizeClassed]uint64

		// arenas是堆arena map。它指向整个可用虚拟地址空间的每个arena帧的堆元数据
		// 使用arenaIndex来计算在这个切片的index
		// 对于Go heap不支持的地址空间区域，arena map可能包含nil
		// 修改被mheap_.lock保护，执行读不需要锁；然而，一个给定
		// 实例在没有持有锁的情况下可以从nil过渡到non-nil。(实例从不
		// 过渡回来为nil)
		// 通常，这是一个2层映射由L1map和可能许多L2maps组成。当他们是大量arena栈时，这样节省空间。
		// 然而，在很多平台(即使是64-bit)，arenaL1Bits等于0，使这成为
		// 一个有效的简单等级的map。在这种场景下，arena[0]永远不会为nil
		arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

		// heapArenaAlloc 是一个块用于申请heapArena对象的预保留。
		// 这只使用在32-bit上，我们预保留这块空间避免和堆本身交叉。
		heapArenAlloc linearAlloc

		// arenaHist是用来添加更多heap arenas的地址列表。它最初由一组通常提示地址填充，伴随这实际堆范围边界增长。
		arenaHints *arenaHint

		// arena 是一块用于申请heap arenas(真实arenas)的预保留空间，只是用在32-bits上。
		arena linearAlloc
		
		// allArenas 是每个映射arena的arenaIndex。它可以用来遍历地址空间
		// 访问由mheap_.lock保护，然而，由于只是追加和老的备份数组从不释放，获取mheap_.lock是安全的，复制切片头，然后释放mheap_.lock。
		allArenas []arenaIdx
	
		// sweepArenas 是发生在标记循环开始的allArenas快照。通过阻塞GC(或者机制抢占)可以安全地读。
		sweepArenas []arenaIdx
		
		// curArena 是heap正在增长的arena。它应该总是物理页对齐的。
		curArena struct {
			base, end uintptr
		}

		// central 空闲列表面对小类型对象
		// padding 确保mcentrals是以CacheLinePadSize字节分隔，所以每个mcentral.lock得到自己的cache line
		// central的索引是spanClass
		central [numSpanClass]struct {
			mcentral mcentral
			pad [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
		}

		// span申请者
		spanalloc fixalloc
		// mcache申请者
		cachealloc fixalloc
		// treapNodes申请者
		treapalloc fixalloc
		// specialfinalizer申请者
		specialfinalizeralloc fixalloc
		// specialprofile申请者
		specialprofilealloc
		// special record 申请者的锁
		speciallock mutex
		// arenaHints 申请者
		arenaHintAlloc fixalloc
		
		// 从不设置，这里只是强制specailfinalizer类型为DWARF
		unused *specialfinalizer

	34 runtime (h *mheap) coalesce(s *mspan)
		// 合并相邻mspan

		// merge 是一个帮助器用来将其他合并到s，删除heap元数据中对这些的引用，然后丢弃他们。这些mspan必须与s相邻
		merge := func(a, b, other *mspan) {
		// 调用者必须确保a.startAddr < b.startAddr 和a或者b是s。a和b必须是相邻的。other是两者中不是s的一个。
		if pageSize < physPageSize && a.scavenged && b.scavenged {
			// 如果我们在pageSize < physPageSize的系统上合并2个回收的spans，他们的边界总是在物理页边界上，因为重组发生在合并过程。
			_, start := a.physPageBounds()
			end, _ := b.physPageBounds()
			if start != end {
				throw("")
			}
		}

		// 通过base和npages调整s，同时也是在heap 元数据中
		s.npages += other.npages
		s.needzero |= other.needzero
		if a == s {
			// s.base()+s.npages*pageSize-1是合并后mspan的尾地址
			h.setSpan(s.base()+s.npages*pageSize-1, s)
		} else {
			s.startAddr = other.startAddr
			h.setSpan(s.base(), s)
		}

		// 大小可能正在改变，所以treap需要删除相邻的对象，同时作为一个联合节点添加回去。
		// 从mheap_.free treap找到mspan所属的treap节点，并移除。 
		h.free.removeSpan(other)
		// 设置该mspan的状态为 mSpanDead
		other.state = mSpanDead
		h.spanalloc.free(unsafe.Pointer(other))
	
		// realign是一个帮助器用于收缩other和扩张s使得他们的边界处于同一个物理页边界上
		realign := func(a, b, other *mspan) {
			//
			if pageSize >= physPageSize {
				return
			}
			h.free.removeSpan(other)
		}

	35 runtime.stackfree(stk stack)
		// go:systemstack
		// stackfree释放stk上的n个字节stack申请。
		// stackfree必须运行在系统栈上因为它用P的资源和必须不能扩张stack

		// 获取当前goroutine
		gp := getg()
		v := unsafe.Pointer(stk.lo)
		// stack的大小
		n := stk.hi - stk.lo
		if n&(n-1) != 0 {
			// 是否是2的幂
			throw("")
		}
		if stk.lo+n < stk.hi {
			throw("")
		}
		fi stackDebug >= 1 {
			println("stackfree", v, n)
			memclrNoHeapPointers(v, n)
		}
		if debug.efence != 0 || stackFromSystem != 0 {
			if debug.efence != 0 || stackFaultOnFree != 0 {
				sysFault(v, n)
			} else {
				sysFree(v, n, &memstats.stacks_sys)
			}
			return
		}
		if msanenables {
			msanfree(v, n)
		}
		
		if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
			order := uint8(0)
			n2 := n
			for n2 > _FixedStack {
				order++
				n2 >> 1
			}
			x := gclinkptr(v)
			c := gp.m.mcache
			if stackNoCache != 0 || c == nil || gp.m.preemptoff != "" {
				lock(&stackpoolmu)
				// 该stack是在heap内申请
				stackpoolfree(x, order)
				unlock(&stackpoolmu)
			} else {
				if c.stackcache[order].size >= _StackCacheSize {
					// 该stack是在heap内申请
					stackcacherelease(c, order)
				}
				x.ptr().next = c.stackcache[order].list
				c.stackcache[order].list = x
				c.stackcache[order].size += n
			}
		} else {
			// 根据地址uintptr(v)获取该地址所属的mspan:s
			s := spanOfUnchecked(uintptr(v))
			// 如果mspan状态不是mSpanManual，则抛出
			if s.state != mSpanManual {
				throw()
			}
			if gcphase == _GCoff {
				// 如果我们正在sweeping，立即释放stack
				osStackFree(s)
				// 手动释放mspan:s
				mheap_.freeManual(s, &memstats.stacks_inuse)
			} else {
				log2npage := stacklog2(s.npages)
				lock(&stackLarge.lock)
				stackLarge.free[log2npage].insert(s)
				unlock(&stackLarge.lock)
			}	
		}

	30 runtime.heapArena struct 
		// heapArena存储heap arena的数据单元。heapArenas储存在Go heap
		// 之外，可以通过mheap_.arenas 索引访问。
		// 它直接通过OS申请，所以理想情况下它应该是系统页的倍数。举例，避免添加小字段。
		
		// bitmap 存储在这个区域的words的指针/标量bitmap，适合heapBits类型可以访问它
		bitmap [heapArenaBitmapBytes]byte
		// spans映射这片区域的虚拟页ID到*mspan，对于申请的spans，它们的页映射span本身。
		// 对于空闲的span，只有最低位和最高位的页映射span本身。内部页映射到任意span。
		// 对于从来没有被申请的页面，spans入口为空 
		// 
		// 修改通过mheap.lock保护。不需要锁可以执行读操作，但是只能执行已知包含在使用或者
		// 栈的span的索引。这就意味着在确定这个地址是会否有效和在spans数组中查找之间肯定不是
		// 安全点。
		spans [pagePerArena]*mspan

		// pageInUse是一个指定那个spans处于mSpanInUse状态的bitmap。这个bitmap的索引是页号
		// 但只是bit对应到每个使用的span的第一页。
		pageInUse [pagePerArena/8]uint8

		// pageMarks 是一个指明那个spans拥有标记对象的bitmap，类似pageInUse,只是每个使用的
		// span的第一页与bit对应。

		// 在标记过程中，写是原子完成。读不是原子性和锁空闲的，因为他们只是在扫描期间发生。(因此和读从不竞争)
		// 这只是用来快速查找这整个span可以释放。
		// 
		pageMarks [pagePerArena/8]uint8



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
	https://blog.csdn.net/u010853261/article/details/103359762
	https://me.csdn.net/u010853261	
