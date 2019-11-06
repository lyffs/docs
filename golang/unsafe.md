## unsafe

-> Go语言的函数传参都是值传递，对形参本身的操作不会影响实参。

-> unsafe包用于Go编译器，在编译阶段使用。

-> 任何类型的指针和unsafe.Pointer可以互相转换，uintptr类型和unsafe.Pointer可以互相转换。

-> uintptr没有指针的语义。而unsafe.Pointer有指针语义。

-> 结构体会被分配一块连续的内存，结构体的地址也代表第一个成员的地址。