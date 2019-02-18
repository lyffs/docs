# APIGW problem #

->1.nf_conntrack: table full, dropping packet

	参考：https://testerhome.com/topics/7509

    netfilter模块加载时的bucket和max配置：

    sudo dmesg | grep conntrack

->2.内核报INFO: task xxxxxx: 634 blocked for more than 120 seconds

	vm.swappiness = 10
	vm.dirty_ratio = 10
	vm.dirty_background_ratio = 5
	vm.dirty_expire_centisecs = 1500
	vm.dirty_writeback_centisecs = 200  

-> 文件打开数设置

	-> 1.转发参数
	文件打开数限制 ulimit -n ,修改/etc/security/limits.conf文件

	*           soft    nofile        655350
    *           hard   nofile        655350	

-> 半连接队列

	在三次握手协议中，服务器维护一个半连接队列，该队列为每个客户端的SYN包开设一个条目（服务端在接收到SYN包的时候，就已经创建了request_sock结构，存储在半连接队列中），该条目表明服务器已接受到SYN包，并向客户端发出确认，正在等待客户的确认包。这些条目所标识的连接在服务器处于SYN_RECV状态，当服务器收到客户的确认包是，删除该条目，服务器进入ESTABLISHED状态。linux提供了几个TCP参数：tcp_syncookies、tcp_synack_retries、tcp_max_syn_backlog、tcp_abort_on_overflow来调整应对。

-> 全连接队列

	当第三次握手时，当server接受到ack包之后，会进入一个新的叫accept队列。
	当accept队列满了之后，即使client继续向server发送ACK的包，也会不被响应，此时ListenOverFlows+1，同时server通过tcp_abort_on_overflow来决定如何返回，0表示直接丢弃该ACK包，1表示发送RST通知client;相应的，client则会分别返回read timeout或者connection reset by peer。另外，tcp_abort_on_overflow是0的话，server过一段时间再次发送syn+ack给client(也就是重新走握手的第二步)，如果client超时等待比较短，就很容易异常了。而客户端收到多个SYN ACK包，则会认为之前的ACK丢包了。于是促使客户端再次发送ACK，在accept对了有空闲的时候最近完成连接。若accept队列始终满员，则最终客户端收到RST包。

	-> 命令

		-> netstat -s | egrep "listen|LISTEN"  查看全队列溢出总数

		-> ss -lnt

		如果State是listen状态，Send-Q表示第三列的listen端口上的全连接队列最大为50，第一列Recv-q为全连接诶队列当前使用了多少。	