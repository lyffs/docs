## CONTEXT

-> 1.context，goroutine的上下文。包含goroutine的运行状态，环境和现场等信息。

-> context主要用来在goroutine之间传递上下文信息，包括：取消信息，超时时间，截止时间，k-v等。

-> 协程的关闭一般会用channel+select方式来控制。

-> context用来解决goroutine之间退出通知，元数据传递的功能。

-> context会在函数间传递。只需要在适当的时候调用cancel函数向goroutines发出取消信息或者调用value函数取出context中的值

-> 创建根节点context的函数。
	->func Backgroud() Context

-> 四个创建子节点context的函数。
	-> func withCancel(parent Context) (ctx Context, cancel CancelFunc)
	-> func WithDeadline(parent Context, deadline time.time) (Context, CancelFunc)
	-> func withTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
	-> func withValue(parent Context, key, val interface{}) Context

-> 	