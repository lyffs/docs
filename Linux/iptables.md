# iptables #

-> 参考：https://www.cnblogs.com/whych/p/9147900.html

-> Iptables服务不是真正的防火墙，只是用来定义防火墙规则功能的防火墙管理工具，将定义好的规则 交由内核中的netfilter（网络过滤器）来读取，从而真正实现防火墙功能。

-> iptable中的规则表是用来容纳规则链。

	-> raw表：确定是否对该数据包进行状态跟踪

	-> mangle表：为数据包设置标记

	-> nat表：修改数据包中的源、目标ip地址或者端口

	-> filter表：确定是否放行该数据包（过滤）

	规则表的先后顺序： raw -> mangle -> nat -> filter
 