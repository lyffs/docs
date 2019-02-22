# network namespace #

-> network namespace 用来隔离网络设备，IP地址，端口等，每个namepsace将会有自己独立的网络栈，路由表，防火墙规则，socket等。

-> 每个新的network namespace默认有一个本地环回接口，除了lo接口外，所有的其他网络设备（物理/虚拟网络接口，网桥等）只能属于一个network namespace。每个socket也只能属于一个network namespace。

-> 当心的network namepsace被创建时，lo接口默认是关闭的，需要自己手动启动起。

-> 