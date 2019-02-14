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