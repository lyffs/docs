# 网络IPC-套接字 #

->
	POSIX.1中指定套接字API是基于4.4BSD套接字接口。套接字描述符在UNIX系统中被当做一种文件描述符。

->
	创建套接字，调用socket函数

	int socket(int domain, int type, int protocol)

	ex:
		socket(AF_INET, SOCK_STREAM, IPPROTO_IP)

	addtion:
		SOCK_RAW套接字提供一个数据包接口，用于直接访问下面的网络层（即因特网中的IP层）。使用这种接口时，应用程序负责构建自己的协议头部，并且需要超级用户的特权。。
->
	字节序是处理器架构特性，用于指示像整数这样过的大数据类型内部的字节如何排序，分为大端和小端。	网络协议指定了字节序，TCP/IPP 协议是使用大端字节序。			

