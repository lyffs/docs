# Slice 学习

## 前沿
	本文是我学习slice的一点心得。

## 详情
	从Go程序反汇编分析Slice

	GO:
	`
	package main
	import "fmt"
	func main() {
		res := make([]int, 0)
		fmt.Println("length:", len(res))
		res = append(res, 1)
		res = append(res, 2)
		res = append(res, 3)
		fmt.Println("length:", len(res))
	}	
	`

	反汇编： go tool objdump -S src > app.s

	ASM: (主要代码)
	`
	LEAQ 0x40(SP), AX
	MOVQ AX, 0x80(SP)
	MOVQ $0x0, 0x88(SP)
	MOVQ $0x0, 0x90(SP)
	...
	MOVQ 0x90(SP), AX
	MOVQ 0x88(SP), CX
	MOVQ 0x80(SP), DX
	LEAQ 0x1(CX), BX
	CMPQ AX, BX
	JG
	...
	LEAQ 0x1068d(IP), SI
	MOVQ SI, 0(SP)
	MOVQ DX, 0x8(SP)
	MOVQ CX, 0x10(SP)
	MOVQ AX, 0x18(SP)
	MOVQ BX, 0x20(SP)
	CALL runtime.growslice(SB)
	`	

### Golang代码
	CALL runtime.growslice(SB)
	`
	func growslice(et *_type, old slice, cap int) slice {
		//前面是否允许竞争检查
		newcap := old.cap
		doublecap := newcap + newcap
		if cap > doublecap {
			newcap = cap 
		} else {
			if old.len < 1024 {
				newcap = doublecap //2倍原来的cap
			} else {
				// Check 0 < newcap to detect overflow
				// and prevent an infinite loop.
				for 0 < newcap && newcap < cap {
					newcap += newcap / 4 // 4/1倍增长
				}
				// Set newcap to the requested cap when
				// the newcap calculation overflowed.
				if newcap <= 0 {
					newcap = cap
				}
			}
		}

		capmem = roundupsize(uintptr(newcap)) //根据类型大小返回内存块大小
		...
		p = mallocgc(capmem, nil, false) //需要切换都g0协程上申请空间
			...
			mp := acquirem() //获取当前工作线程
			...
			c := gomcache() //返回当前个工作线程的cache(缓存)
			...
			if size <= maxSmallSize(32768) {
				if noscan && size < maxTinySize(16) {
					// Tiny allocator, be noscan (don't have pointers)
					...
					if off+size <= maxTinySize && c.tiny != 0 {
						x = unsafe.Pointer(c.tiny + off)
						releasem(mp)
						...
						return x
					}
					// Allocate a new maxTinySize block. //申请一个新的maxTinySize（16bytes）的内存块
						
				} else {
					...
					v := nextFreeFast(span)
					...
					if v == 0 {
						v, span, shouldhelpgc = c.nextFree(spc)
					}
					...
					if needzero && span.needzero != 0 {
						memclrNoHeapPointers(unsafe.Pointer(v), size) //是由汇编实现
					}
				}
			else {
				systemstack(func() {
					s = largeAlloc(size, needzero, noscan)
				})	//systemstack 由汇编实现，切换到
			}	
		}

	func mstart() {

	}

	`

### 汇编代码
	TEXT runtime·systemstack(SB), NOSPLIT, $0-8	//无函数栈，1个参数无返回值
		MOVQ    fn+0(FP), DI    // DI = fn，将第一个参数放在DI寄存器
		get_tls(CX) // get_tls 是宏，获取当前协程
		MOVQ    g(CX), AX       // AX = g，将g存在AX寄存器
		MOVQ    g_m(AX), BX     // BX = m，将工作线程存在BX寄存器
		CMPQ    AX, m_gsignal(BX) //当前工作线程的gsignal是当前协程
		JEQ     noswitch //相等无需切换

		MOVQ    m_g0(BX), DX    // DX = g0，获取g0的协程
        CMPQ    AX, DX  //相等无需切换
        JEQ     noswitch

        // switch stacks
        // save our state in g->sched. Pretend to
        // be systemstack_switch if the G stack is scanned.
        MOVQ    $runtime·systemstack_switch(SB), SI
        MOVQ    SI, (g_sched+gobuf_pc)(AX) //暂存状态
        MOVQ    SP, (g_sched+gobuf_sp)(AX)
        MOVQ    AX, (g_sched+gobuf_g)(AX)
        MOVQ    BP, (g_sched+gobuf_bp)(AX)

         // switch to g0
        MOVQ    DX, g(CX)
        MOVQ    (g_sched+gobuf_sp)(DX), BX
        // make it look like mstart called systemstack on g0, to stop traceback
        SUBQ    $8, BX
        MOVQ    $runtime·mstart(SB), DX
        MOVQ    DX, 0(BX)
        MOVQ    BX, SP

        // call target function
        MOVQ    DI, DX
        MOVQ    0(DI), DI
        CALL    DI   

        // switch back to g
        get_tls(CX)
        MOVQ    g(CX), AX
        MOVQ    g_m(AX), BX
        MOVQ    m_curg(BX), AX
        MOVQ    AX, g(CX)
        MOVQ    (g_sched+gobuf_sp)(AX), SP
        MOVQ    $0, (g_sched+gobuf_sp)(AX)
        RET

        noswitch:
        // already on m stack; tail call the function
        // Using a tail call here cleans up tracebacks since we won't stop
        // at an intermediate systemstack.
        MOVQ    DI, DX
        MOVQ    0(DI), DI
        JMP     DI

bad:
        // Bad: g is not gsignal, not g0, not curg. What is it?
        MOVQ    $runtime·badsystemstack(SB), AX
        CALL    AX
        INT     $3



	
## 参考资料
https://github.com/JerryZhou/golang-doc