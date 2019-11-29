# Golang汇编学习

## 前沿
本文是我学习Go汇编的一点心得

## 详情

### 场景
	1.）算法加速，golang编译器生成的机器码基本上都是通用代码，优化程度一般，我们需要用到特殊优化逻辑、特殊的CPU指令让我们的算法运行速度更快。https://github.com/minio/sha256-simd
	2.）摆脱golang编译器一些约束，如通过汇编调用其他package的私有函数。https://sitano.github.io/2016/04/28/golang-private/
	3.）进行一些hack的事，通过汇编适配其他语言的ABI来直接调用其他语言的函数。 https://github.com/petermattis/fastcgo
	4.）利用 //go:noescape 进行内存分配优化，golang编译器拥有逃逸分析，用于决定每一个变量是分配在堆内存、还是函数栈。https://github.com/golang/go/blob/d1fa58719e171afedfbcdf3646ee574afc08086c/src/reflect/value.go#L2585-L2597

#### Go几个虚拟寄存器

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
	其实伪寄存器FP和SP相当于plan9伪汇编中的一个助记符，他们是根据当前函数栈空间计算出来的一个相对于物理寄存器SP的一个偏移量坐标。当在一个函数中，如果用户手动修改了物理寄存器SP的偏移，则伪寄存器FP和SP也随之发生对应的偏移。

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

##### 调用其他函数
	在汇编中调用其他函数通常是：
	JMP： 直接跳转时，与函数栈空间相关的几个寄存器SP/FP不会发生变化，可以理解为被调用者复用调用者的栈空间。此时，参数传递采用寄存器传递，调用者和
		被调用者协商好使用那些寄存器传递参数，调用者将参数写入这些寄存器，然后跳转到被调用者，被调用者从相关寄存器读出相关参数。

	CALL: 通过CALL命令来调用其他函数时，占空间会发生响应变化（寄存器SP/FP随之发生变化）。传递参数时，我们需要输入参数、返回值按之前将的栈布局安排在调用者的栈顶
	（低地址），然后在调用CALL命令来调用其他函数，调用CALL命令后，SP寄存区会下移一个WORD，然后进入新函数的栈空间运行。return addr（函数返回地址）不需要用户维护，CALL
	指令会自动维护。

##### 回调函数/闭包
	当函数参数中包含回调函数时，回调函数的指针通过一种简介方式传入，之所以采用这种设计也是为例照顾闭包调用的实现。在Golang的ABI中，关于回调函数、闭包的
	上下文由调用者来维护，被调用者按照规定的格式来使用。	

	1.调用者需要申请一段临时内存区域来存储函数指针，当传递参数是闭包时，该临时内存区域开可以进行扩充，用于存储闭包中捕获的变量，通常编译器将该内存区域定义成结构体：
	struct { f uintptr; a *int}的结构。该临时内存区域可以分配在栈上，也可以分配在堆上，也可以分配到寄存器上，到底分配到那里，需要编译器根据逃逸分析的结果来决定。
	2.将临时内存区域的地址存储于对应 被调用函数 入参的对应位置上；其他参数按照上面常规方法放置。
	3.使用CALL执行调用 被调用函数（callee-call）
	4.在被调用函数（callee-call） 中从对应参数位置中去除临时内存区域的指针存储在指定寄存器DX(仅限于AMD64平台)
	5.然后从DX指向的临时内存区域的首部取出函数指针，存储于AX
	6.然后在执行CALL AX指令来调用传入的回调函数。
	7.当回调函数是闭包时，需要使用捕获的变量时，直接通过寄存器DX加对应偏移量来获取。		 		

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

	MOVQ命令，在寄存器加偏移的情况下MOVQ会对地址进行解引用。
	MOVQ (AX), BX   // => BX = *AX 将AX指向的内存区域8byte赋值给BX
	MOVQ AX, BX     // => BX = AX 将AX中存储的内容赋值给BX，注意区别

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

##### 

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
	汇编命令：go tool compile -l -N -S
	反汇编命令：go tool objdump -S
	

## 参考资料
	https://www.altoros.com/blog/golang-internals-part-1-main-concepts-and-project-structure/
	https://github.com/yangyuqian/technical-articles/blob/master/asm/golang-plan9-assembly-cn.md
	https://xargin.com/plan9-assembly/
	https://www.cnblogs.com/landv/p/11589074.html
	https://quasilyte.dev/blog/post/go-asm-complementary-reference/#external-resources
	https://www.cnblogs.com/landv/p/11589074.html
	http://blog.studygolang.com/2013/05/asm_and_plan9_asm/
	https://zhuanlan.zhihu.com/p/56750445?utm_campaign=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com
	https://lrita.github.io/2017/12/12/golang-asm/#why
	https://blog.gopheracademy.com/advent-2016/peachpy/
	https://sitano.github.io/2016/04/28/golang-private/
	https://syslog.ravelin.com/anatomy-of-a-function-call-in-go-f6fc81b80ecc


