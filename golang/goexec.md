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
	2.runtime.check()

	//
	3.runtime.args()

	//
	4.runtime.osinit()
		ncpu = getproccount() //获取CPU计算
		physHugePageSize = getHugePageSize() //获取大页大小
		
	//The bootstrap sequence is:
	// call osinit
	// call schedinit
	// make & queue new G
	// call runtime.mstart

	5.runtime.schedinit()
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

	6.runtime.mallocinit()
		mheap_.init() //堆初始化
		_g_ := getg()
		_g_.m.mcache = allocmacache()

	7.runtime.mcommoninit()
		mpreinit(mp) //m.gsignal初始化，其中新建g，同时g的stack需要申请。从系统内存申请，或者从stackpool申请，或者从heap（mheap_）申请。
		
	8.runtime.procresize(nproc int32) *p
		// initialize new P's
		for i := old; i < nprocs; i++ {
			pp := allp[i]
			if pp == nil {
				pp = new(p)
			}

			pp.init(i) //设置pp的状态为:_Pgcstop,pp.mcache = getg().m.mcache //或者pp.mcache = allocmcache()
			atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
		}

	9.runtime.main()
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

	10.runtime.gcenable() 
		c := make(chan int, 2)
		go bgsweep(c)
		go bgscavenge(c)
		<-c
		<-c
		memstats.enablegc = true

	11.func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int)
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

		
	12.runtime.newproc(siz int32, fn *funcval) 
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

		
	13.runtime.park_m
		// 在g0中继续暂停
		
		_g_ := getg()
		
		//将gp状态从_Grunning变更为_Gwaiting
		casgstatus(gp, _Grunning, _Gwaiting)

		//
		dropg()

	14.runtime.casgstatus(gp *g, oldval, newval uint32)
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

	15.runtime.dropg()
		// dropg将移除m和当前routine m->curg的关联
		// 通常调用者变更gp的Grunning状态然后立刻调用dropg完成该项工作
		// 调用者还负责安排gp在恰当的时间状态变更为ready重新启动。
		// 在调用dropg和稍后安排gp状态变更为ready，调用者可以做其他工作
		// 但最终应该调用schedule来重新启动这个m上面的goroutine调度。

		_g_ := getg()

		 setMNoWB(&_g_.m.curg.m, nil) //no写屏障设置m当前gouroutine的m为空
		 setGNoWB(&_g_.m.curg, nil) //no写屏障设置m当前goroutine为空


### 参考
	https://docs.oracle.com/cd/E19205-01/820-1200/blaoy/index.html

