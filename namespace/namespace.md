# NameSpace #

-> USER

	-> 同样一个用户的USER ID 和 GROUP ID 在不同的USER NAMESPACE中可能不一样（与PID NAMESPACE类似）。换一句话说，就是在一个USER NAMESPAC中是普通用户，在另外一个 USER NAMESPACE中可能是root用户。 除了系统默认USER NAMESPACE外，所有的USER NAMESPACE都有一个父 USER NAMESPACE。在一个进程中，调用unshare或者clone 创建新的USER NAMESAPCE时，当前进程的USER NAMESPACE为父 USER NAMESPACE，新的USER NAMESPACE为子USER NAMESPACE。

	-> 当新的USER NAMESPACE 创建并映射好uid、gid之后，这个USER NAMESPACE的第一个进程拥有完整的所有 capabilities,意味着它就可以创建其他类型的namespace

	-> 当用子USER NAMESPACE用户访问父 USER NAMESPACE的资源时，它启动的进程capability都为空，所以子USER NAMESPACE的root用户在父USER NAMESPACE中相当于一个普通用户。

	-> Linux下每个namespace，都有一个user namespace与之关联，这个user namespace就是创建对应namespace时进程所属的USER NAMESPACE，相当与每个namespace都有一个owner(USER NAMESPACE)，这样就可以保证对任何NAMESPACE的操作都受到USER NAMESPACE权限控制。