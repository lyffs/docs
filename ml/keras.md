### Keras

->	Keras是一个用python编写的高级神经网络API.

->	模型
	主要有两类主要的模型：sequential顺序模型和使用函数式API的Model类模型。

->	Sequential模型
	顺序模型是多个网络层的线性叠加，可以将网络层实例的列表传递给Sequential构造器，来创建一个Sequential模型。
	->	指定输入数据的尺寸
		传递一个input_shape参数给第一个层。在input_shape中不包含数据的batch大小
			

