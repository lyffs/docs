# Golang汇编学习

## 前沿
本文是我学习Go汇编的一点心得

## 详情

### Go几个虚拟寄存器

#### 栈调整
	栈的调整是通过对硬件SP寄存器进行运行来实现。

#### 数据搬运
	场数在plan9汇编用$num表示，可以为负数，默认情况下为十进制。搬运的长度是有MOV后缀决定的。
	MOVB $1,DI // 1 byte
	MOVW $0x10, BX // 2bytes
	MOVD $1, DX // 4bytes
	MOVQ $-10, AX // 8bytes

#### 条件跳转
	JMP addr //跳转到地址，地址可为代码中地址
	JMP label //跳转到标签，可以跳转到同一函数内的标签位置
	JMP 2(PC) //以当前指令为基础，向前/向后跳转X行
	JMP -2(PC) //同上
	JNZ target //如果zero flag被set过，则跳转

#### 指令集
	参考源代码arch 部分：https://github.com/golang/arch/blob/master/x86/x86.csv

#### 通用寄存器
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

#### 伪寄存器
	GO汇编引入4个伪寄存器。
	FP: Frame poniter: arguments and locals
	PC: Program counter: jumps and branches
	SB: Static base pointer: global symbols
	SP: Stack pointer: top of stack

	FP: 使用形如symbol+offset（FP）的方式，引用函数的输入参数。例如args0+0(FP)
	PC: 就是体系结构中的PC寄存器
	SB: 全局静态基指针，一般用来声明函数或者全局变量。
	SP: 指向当前栈帧的局部变量的开始位置。使用形如symbol+offset(SP)方式，引用函数局部变量。offset的取值范围：[-framesize,0)

	go tool objdump/go tool compile -S 输出的代码是没有伪SP和FP寄存器的。在汇编结果中，只有真实的SP寄存器。

#### 变量声明
	在汇编所谓的变量，一般是存储在 .rodata或者 .data段中的只读值。对应到应用层的话，就是已初始化的全局的const，var，statics常量/变量。

#### 函数声明
	在plan9中TEXT是一个指令，用来定义一个函数。	

	TEXT package·add(SB),NOSPLIT,$32-32
	 		|     |                | |
	 		包名  函数名           栈帧大小 参数及返回值大小

#### 栈结构

		---------------------------------
		current func arg0
		--------------------------------- <-------FP (pseudo FP)
		caller ret addr
	---	---------------------------------
	 |	caller BP(*)
	 |	--------------------------------- <-------SP (pseudo SP, 实际上是当前栈帧的BP位置)
	 |	local var0
	 |	---------------------------------
	 |	local var1
	 |	---------------------------------
	 s	local var2
	 t	---------------------------------
	 a	.........
	 c	---------------------------------
	 k	local varN
	 	---------------------------------
	 f	temporarilly
	 r	unused space
	 a	---------------------------------
	 m	call retn 调用函数第n个返回值
	 e	--------------------------------
		call ret(n-1) 调用函数第n-1个返回值
	 |	--------------------------------
	 |	..............
	 |	--------------------------------
	 |	call ret1 调用函数第1个返回值
	 |	--------------------------------
	 |	call argn 调用函数第n个参数
	 |	--------------------------------
	 |	.....................
	 |	--------------------------------
	 |	call arg1 调用函数第一个参数
	 |	-------------------------------- <--------- 硬件SP位置
	 |	return addr 返回值地址
	---	--------------------------------

		栈是朝低地址方向增长的。

##### argsize和framesize计算规则
	如函数申明：TEXT package·add(SB),NOSPLIT,$16-32
	$16-32表示 $framesize-argsize。Go在函数调用时，参数和返回值都需要由caller 在其栈帧上备好空间。argsize计算方法是：参数大小求和+返回值大小求和。如入参是3个int64类型，返回值是1个int64类型，那么这里的argsize=sizeof(int64) * 4。
	



## 参考资料

   参考资料：https://xargin.com/plan9-assembly/