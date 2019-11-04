1.Map
	-> 参考：https://mp.weixin.qq.com/s/2CDpE5wfoiNXm1agMAq4wA

	-> 最主要的数据结构：哈希查找树、搜索树。

	-> 哈希查找表 用一个哈希函数将key分配到不同的桶（bucket，也就是数组的不同index）。 会遇到碰撞的问题，解决的方法包含：链表法和开放地址法。

	-> 链表法用链表实现bucket;开放地址法通过一定规律（再次hash），选择新的bucket。

	-> 搜索树一般采用自平衡搜索树，包括AVL树和红黑树。

	-> GO语言采用哈希查找表，并且使用链表解决冲突的实现MAP

	-> BMap就是我们常说的桶，桶里面最多装8个key，这些key之所以落入同一个桶，是因为他们经过哈希计算，结果是一类的。有根据hash值的高8位来决定key到底落入桶内的那个位置。

	-> makeMap和makeSlice的区别是：当map和slice作为函数参数时，在函数参数内容部对map的操作会影响map自身；而对slice则不会。主要原因是：一个是指针，一个是结构体。

	-> 计算哈希值时，加入hash0引入随机性。

	->	key定位公式：
		k:=add(unsafe.Pointer(b),dataOffset+i*uintptr(t.keysize))

	->	value定位公式：
		v:=add(unsafe.Pointer(b),dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))	

-> 内存泄漏
	https://cloud.tencent.com/developer/article/1437506		