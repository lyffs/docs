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

###
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

		capmem = roundupsize(uintptr(newcap))
		p = mallocgc(capmem, nil, false) //需要切换都g0协程上申请空间

	}
	`
		
