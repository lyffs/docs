# SPARK ML #

-> DOCS: http://spark.apache.org/docs/1.2.2/ml-guide.html

-> Main Concepts

	-> ML Dataset

	-> Transformer

	-> Estimator

	-> Pipeline

	-> Param


-> learning algorithm

	-> LogisticRegression //逻辑回归，用来解决分类问题。回归于分类区别在于：回归所预测的目标量是取值是连续的；而分类锁预测的目标变量的取值是离散的。

-> pipeline

	A pipeline is specified as a sequence of stages, and each stage is either a Transformer or An Estimator. These stages are run in order, and the input dataset is modified as is passes through each stage.For Transformer stages, the transform() method is called on the dataset.	For Estimator stages, the fit() method is called to produce a Transformer
