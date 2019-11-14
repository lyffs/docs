## IO多路复用 netpoll

参考：https://taohuawu.club/go-netpoll-io-multiplexing-reactor
		https://ninokop.github.io/2018/02/18/go-net/
		


-> 内核一旦发现进程指定的一个或者多个I/O条件（文件描述符的I/O条件）就绪，内核就通知进程，这个能力成为I/O复用。

-> 实现：select/poll/epoll

-> select/poll调用时会将全部监听的fd集合从用户态拷贝至内核态并线性扫描一遍找出就绪的FD在返回到用户态。

-> epoll_wait 实际上就是去检查rd_list双向链表中是否有就绪的FD。

-> 