## Memory 

### 简介
	Golang内存管理基于tcmalloc。

### 详情

#### 逃逸分析
	Golang编译器决定变量应该分配到什么地方时进行逃逸分析。如果编译不能确保变量在函数return之后不再被引用，编译器就会将变量分配到堆上。而且，如果一个局部变量非常大，那么它也应该被分配到堆上不是栈上。

#### 关键数据结构
	mcache: per-P cache，可以认为是local cache
	mcentral: 全局cache, mcache不够用时向mcentral申请
	mheap: 当mcentral不够用时，通过mheap向操作系统申请。

##### mspan
	span在tcmalloc中作为一种管理内存的基本单位而存在。
	sizeclass: 0 ~ _NumSizeClasses之间的值。如sizeclass=3，那么这个span被分割成32byte的块
	elemsize: 块大小。如sizeclass=3，elemsize=32byte
	freeindex: 0 ~ nelemes-1，表示分配到第几个块。

##### mcache
	

##### mcentral
	当mcache不够用，会从mcentral申请。
	mcentral没有包含在mheap中。


##### mheap
	allspans []*mspan: 所有的spans都是通过mheap_申请，所有申请过mspan都会记录在allspans。
	central [_numsizeclasses]: pad可以认为是一个字节填充，为了避免伪共享问题（false sharing）。
	sweepgen,sweepdone: GC相关。
	freelarge: mSpanList:	mspan组成的链表，每个元素（mspan）的page个数大于127。
	spans []*mspan:	记录arena区域页号（page number）和mspan的映射关系。
	arena_start,arena_end,arena_used：	arena是Golang中用于分配内存的连续虚拟地址区域。由mheap管理，堆上申请的所有内存都来自arena。内存布局：
	+-----------------------+---------------------+-----------------------+
	|    spans              |    bitmap           |   arena               |
	+-----------------------+---------------------+-----------------------+
	spanalloc,cachealloc fixalloc:	fixalloc是free-list，用来分配特定大小的块。

###### arena
	bitmap用两个bit表示一个字的状态。
	spans记录arena区块页号和对应mspan指针的对应关系。



### 参考
	http://legendtkl.com/2017/04/02/golang-alloc/
	http://goog-perftools.sourceforge.net/doc/tcmalloc.html

