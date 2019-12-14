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
		mallocinit() //内存管理初始化

	6.runtime.mallocinit()
		mheap_.init() //堆初始化
		_g_ := getg()
		_g_.m.mcache = allocmacache()

		
		
		

			
	
		


	
