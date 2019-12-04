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