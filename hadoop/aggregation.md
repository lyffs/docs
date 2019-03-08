# 聚合 #

->	参考：https://blog.csdn.net/yu616568/article/details/72597759

->	聚合操作是 OLAP分析查询中最常见操作，它对应的SQL中的GROUP BY字句，OLAP中的上卷下钻操作无非就是对于GROUP BY和WHERE条件的变化。

->	从OLAP引擎来划分的话主要分成MOLAP和ROLAP,MOLAP基于CUBE模型进行预计算；ROLAP则始终基于原始数据计算。

->	典型的聚合计算包括：COUNT/SUM/MAX/MIN/COUNT DISTINCT等等

->	聚合操作伪代码：

```
	func Map(mmap *map[string]int, key string) {
	if mmap == nil {
		return
	}
	if value, exist := (*mmap)[key]; !exist {
		(*mmap)[key] = 1
	} else {
		(*mmap)[key] = value+1
	}
}
```

->	