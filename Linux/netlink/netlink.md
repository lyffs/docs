# netlink #

-> linux 用户态与内核态交流的主要方式之一。它的通信依据是一个对应于进程的标识，一般定位该进程的ID。AF_NETLINK族套接字像一个连接用户空间和内核的双工管道。

-> AF_NETLINK socket。先生成所需套接字，并绑定一个sockaddr结构。sockfd = socket(AF_NETLINK，socket_type，netlink_family);bind(sockfd, (struct sockadd*)&nl，sizeof(nl))。


