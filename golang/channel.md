## channel

-> goroutine和channel是GO语言并发编程的两大基石。Goroutine用于执行并发任务，channel用于goroutine之间的同步、通信。

-> CSP模型： 不要通过共享内存来通信，而要通过通信来实现共享内存。

-> 对Chan的发送和接受操作都会在编译期间转换成底层的发送接收函数。