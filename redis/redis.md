# Redis #

-> Redis 是一个内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。

-> 支持的数据结构：

	-> 字符串(string)、散列(hashes)、列表(lists)、集合(sets)、有序集合(sorted sets)

	->	字符串：

		->	二进制安全字符串

		-ex:

			->	set mykey somevalue

			->	get mykey
			
			->	incr mykey

			->	getset (设置新值并返回原值)

			->	expire mykey
			
			->	ttl mykey

	->	列表：

		->	按插入顺序安排序的字符串元素的集合。基本上就是链表

	->	集合：

		->	不重复但无序的字符串元素集合

	->	有序集合：
	
		->	类似	集合，每个字符串元素都会关联一个叫score浮动数值。里面的元素都是通过score进行排序

		->	ex:

			->	ZRANGEBYSCOPE

			-> 	ZREMRANGEBYSCOPE

			->	ZREVRANGEBYSCOPE

			->	-inf +inf

	->	散列：

		->	有field和关联的value组成的map。field和value都是字符串的。


->	事务

	->	MULTI、EXEC、DISCARD、WATCH

->	持久化

	->	RDB和AOF

		-> RDB持久化方式能够在指定时间间隔对你的数据进行快照储存。

		-> AOF持久化方式记录每次对服务器写操作。AOF命令以REDIS协议追加保存每次写操作到文件末尾。

	->	RDB
	
		-> 优点：RDB时候一个非常紧凑文件，它保存某个时间点的数据集，非常适用于数据集的备份

		-> 缺点：数据丢失量多

	->	AOF	

		->	优点：Redis更耐用，丢失数据量更少。可以使用不同的fsync策略： 无fsync和秒级fsync。写入操作都是以REDIS协议格式保存。RESP协议特点：简单，易于分析。

		->	缺点：储存文件比较大，速度可能慢于RDB



