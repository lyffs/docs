### Golang汇编学习

## 前沿
本文是我学习Go汇编的一点心得

## 详情

# Go几个虚拟寄存器

-> 栈调整
	栈的调整是通过对硬件SP寄存器进行运行来实现。

-> 数据搬运
	场数在plan9汇编用$num表示，可以为负数，默认情况下为十进制。搬运的长度是有MOV后缀决定的。
	MOVB $1,DI // 1 byte
	MOVW $0x10, BX // 2bytes
	MOVD $1, DX // 4bytes
	MOVQ $-10, AX // 8bytes

-> 条件跳转
	JMP addr //跳转到地址，地址可为代码中地址
	JMP label //跳转到标签，可以跳转到同一函数内的标签位置
	JMP 2(PC) //以当前指令为基础，向前/向后跳转X行
	JMP -2(PC) //同上
	JNZ target //如果zero flag被set过，则跳转

-> 指令集
	参考源代码arch 部分：https://github.com/golang/arch/blob/master/x86/x86.csv

-> 通用寄存器
	rax -> AX
	rbx -> BX
	rcx -> CX
	rdx -> DX
	rdi -> DI
	rsi -> SI
	r8 ~ r15 -> R8 ~ R15
	RBP -> bp
	RSP -> sp

	bp和sp用来管理栈顶和栈底，最好不要拿来运算。plan9中使用寄存器不需要带r或者e的前缀，例如rax，只要写成AX即可。

-> 伪寄存器
	GO汇编引入4个伪寄存器。
	FP: Frame poniter: arguments and locals
	PC: Program counter: jumps and branches
	SB: Static base pointer: global symbols
	SP: Stack pointer: top of stack



## 参考资料

-> -> 参考资料：https://xargin.com/plan9-assembly/