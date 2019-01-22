# 网络IPC-套接字 #

->
	POSIX.1中指定套接字API是基于4.4BSD套接字接口。套接字描述符在UNIX系统中被当做一种文件描述符。

->
	创建套接字，调用socket函数

	int socket(int domain, int type, int protocol)

	ex:
		socket(AF_INET, SOCK_STREAM, IPPROTO_IP)	

