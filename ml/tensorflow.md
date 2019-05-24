### TensorFlow

-> 入门
	升级pip pip install --upgrade pip
	通过pip安装tf pip install https://storage.googleapis.com/tensorflow/linux/cpu/tensorflow-0.5.0-cp27-none-linux_x86_64.whl

->	基本概念
	使用图(graph)表示计算任务
	在被称为会话(session)的上下文中执行图
	使用tensor表示数据
	通过变量维护状态
	使用feed和fetch可以为任意的操作赋值或者从其中获取数据

->	Tensorflow程序通常被组织成一个构建阶段和一个执行阶段。在构建阶段，op的执行步骤被描述成一个图。在执行阶段，使用会话执行图中的op。	


->	数据结构
	Tensorflow用张量这种数据结构来表示所有的数据，你可以把一个张量想象成一个n维数组或列表。


-> 高级API
	Keras
		