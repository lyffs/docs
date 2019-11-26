# Golang汇编学习

## 前沿
本文是我学习Go汇编的一点心得

## 详情

### Go几个虚拟寄存器

#### 栈调整
	栈的调整是通过对硬件SP寄存器进行运行来实现。
	如：
	SUBQ $0x18, SP //为函数分配函数栈帧
	ADDQ $0x18, SP //对SP做加法，清除函数栈帧

#### 数据搬运
	常数在plan9汇编用$num表示，可以为负数，默认情况下为十进制。搬运的长度是有MOV后缀决定的。
	MOVB $1,DI // 1 byte
	MOVW $0x10, BX // 2bytes
	MOVD $1, DX // 4bytes
	MOVQ $-10, AX // 8bytes

#### 条件跳转
##### section跳转
	JMP addr //跳转到地址，地址可为代码中地址
	JMP label //跳转到标签，可以跳转到同一函数内的标签位置
	JMP 2(PC) //以当前指令为基础，向前/向后跳转X行
	JMP -2(PC) //同上
	JNZ target //如果zero flag被set过，则跳转

##### 函数调用跳转
	JMQ SP/BP 不会发生变化，栈空间不会发生变化。
	CALL 栈空间会发生响应的变化，传递参数时，我们需要输入参数，返回值按之前将栈的布局安排在调用者的栈顶，然后在调用CALL函数来调用其函数，
	调用CALL命令后，SP寄存器会下移一个WORD，然后进入新函数的栈空间执行。
	


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
	使用DATA结合GLOBL来定义一个变量。DATA用法如下：
	DATA symbol+offset(SB)/width, value

	offset的函数是该值相对于符号symbol的偏移，而不是相对于全局某个地址的偏移。
	使用GLOBL指令来将变量声明为global，额外接收2个参数，一个是flag，另外一个是变量的总大小。
	GLOBL divtab(SB), RODATA, $64

	定义数据，或字符串，这是需要用到非0的offset，例如：
	DATA bio<>+0(SB)/8, $"oh yes i"
	DATA bio<>+8(SB)/8, $"am here"
	GLOBL bio<>(SB), RODATA, $16

	go编译器为了方便汇编访问struct的指定字段，会在编译过程中自动生成一个go_asm.h文件，可以通过#include "go_asm.h" 语言来引用，该文件中会生成该包内全部struct的每个字段的偏移量宏定义与结构体大小到的宏定义。

	我们可以通过命令go tool compile -S -asmhdr dump.h *.go来导出相关文件编译过程中会生成的宏定义

#### 函数声明
	在plan9中TEXT是一个指令，用来定义一个函数。	

	TEXT package·add(SB),NOSPLIT,$32-32
	 		|     |                | |
	 		包名  函数名           栈帧大小 参数及返回值大小

	 当有NOSPLIT表示时，可以不写输入参数，返回值占用的大小（这时会强行插入CALLER BP）		

#### 栈结构

		---------------------------------
		current func arg0
		--------------------------------- <-------FP (pseudo FP)
		caller ret addr
	---	--------------------------------- high address
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
	---	-------------------------------- low address

		栈是朝低地址方向增长的。

##### argsize和framesize计算规则
	如函数申明：TEXT package·add(SB),NOSPLIT,$16-32
	argsize:caller（存储）
	framesize:callee（存储）
	`$16-32表示 $framesize-argsize。Go在函数调用时，参数和返回值都需要由caller再起栈帧上备好空间`。argsize计算方法是：参数大小求和+返回值大小求和。如入参是3个int64类型，返回值是1个int64类型，那么这里的argsize=sizeof(int64) * 4。
	return address(rip)的值也是存储在caller的stack frame上的，但是这个过程是由CALL指令和RET指令完成PC寄存器的保存和恢复。	

##### 地址运算
	地址运算也是用LEA 指令，英文愿意为 Load Effective Addresss，amd64平台地址都是8个字节，所有直接就用LEAQ就好。
	例子： LEAQ (BX)(AX*1), CX

##### 示例
	math.go
	`
	package main
	import "fmt"
	func add(a, b int)int //汇编声明
	func main() {
		fmt.Println(add(10,11))
	}
	`


	math.s
	`
	#include "textflag.h"

	TEXT .add(SB), NOSPLIT, $0-24
		MOVQ a+0(FP), AX
		MOVQ b+8(FP), BX
		ADDQ BX, AX
		MOVQ AX, ret+16(FP)
		RET
	`
	
	把两个文件放在任意目录下，执行go build并运行就可以看到效果

##### 数据结构

###### 数据类型
	标准库的数值类型：
	1.int/int8/int16/int32/int64
	2.uint/uint8/uint16/uint32/uint64
	3.float32/float64
	4.byte/rune
	5.uintpr

	这些类型在汇编中就是一段储存着数据的连续内存，只是内存长度不一样，操作的时候看好数据长度就行。

	1.struct
	struct在汇编层面实际上就是一段连续内存，在作为参数传给函数时，会将其展开在caller的栈上传给对应的callee:


##### 命令
	

## 参考资料

	https://xargin.com/plan9-assembly/
	https://www.cnblogs.com/landv/p/11589074.html
	https://quasilyte.dev/blog/post/go-asm-complementary-reference/#external-resources
	https://www.cnblogs.com/landv/p/11589074.html
	http://blog.studygolang.com/2013/05/asm_and_plan9_asm/
	https://zhuanlan.zhihu.com/p/56750445?utm_campaign=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com
	https://lrita.github.io/2017/12/12/golang-asm/#why
