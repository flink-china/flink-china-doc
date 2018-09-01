# 多元线性回归

* [描述](#描述)
* [操作](#操作)
    * [拟合](#拟合)
    * [预测](#预测)
* [参数](#参数)
* [示例](#示例)

## 描述

多元线性回归尝试找到最适合所提供输入数据的线性函数。只要给定赋值了的(\mathbf{x}, y)的输入数据的集合，多元线性回归找到向量\mathbf{w}，这样残差平方的总和最小化可以表示为：

$$ S(\mathbf{w}) = \sum_{i=1} \left(y - \mathbf{w}^T\mathbf{x_i} \right)^2 $$

用矩阵表示法可以这样写：

$$ \mathbf{w}^* = \arg \min_{\mathbf{w}} (\mathbf{y} - X\mathbf{w})^2 $$

这个问题有一个封闭的形式解决方案，由下式给出：

$$ \mathbf{w}^* = \left(X^TX\right)^{-1}X^T\mathbf{y} $$

然而，在输入数据集如此庞大以至于对整个数据集完成解析是不太可能的情况下，可以应用随机梯度下降（SGD）来近似解决方案。 SGD首先针对输入数据集的随机子集计算梯度。给定点$$ \mathbf{x}_i $$的梯度由下式给出：

$$ \nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i}) = 2\left(\mathbf{w}^T\mathbf{x_i} -
    y\right)\mathbf{x_i} $$

梯度被平均和缩放。缩放由\gamma = \frac{s}{\sqrt{j}} 定义，其中s是初始步长，`j`是当前迭代次数。从当前权重向量中减去得到的梯度，为下一次迭代提供新的权重向量：

$$ \mathbf{w}_{t+1} = \mathbf{w}_t - \gamma \frac{1}{n}\sum_{i=1}^n \nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i}) $$

多元线性回归算法基于动态收敛准则来计算固定数量的SGD迭代或终止。收敛标准是残差平方和的相对变化：

$$ \frac{S_{k-1} - S_k}{S_{k-1}} < \rho $$
## 操作

多元线性回归是一个预测变量。因此，它支持拟合和预测操作


### 拟合

多元线性回归在一组`LabeledVector上`训练：

*   `fit: DataSet[LabeledVector] => Unit`

### 预测

多元线性回归为向量的所有子类型预测相应的回归值：

*   `predict[T <: Vector]: DataSet[T] => DataSet[(T, Double)]`

## 参数

MinMax Scaler 可由下列两个参数进行控制：

参数 | 描述
---|---
**Iterations** | 最大迭代次数。 （Default value:**10**）
**Stepsize** | 梯度下降法的初始步长。该值控制梯度下降方法在梯度相反方的向上移动的距离。调整此参数可能对于使其稳定并获得更好的性能至关重要。 （Default value:**0.1**）
**ConvergenceThreshold** | 残差平方和的相对变化的阈值，直到迭代停止。 (Default value: **None**) 
**LearningRateMethod** | 学习率方法用于计算每次迭代的有效学习率。请参阅提供的学习速率方法列表。 （Default value:**LearningRateMethod.Default**）


## 示例

```scala
// 创建多元线性回归学习器
val mlr = MultipleLinearRegression()
.setIterations(10)
.setStepsize(0.5)
.setConvergenceThreshold(0.001)

// 获取训练和测试数据集
val trainingDS: DataSet[LabeledVector] = ...
val testingDS: DataSet[Vector] = ...

// 使线性模型拟合所提供的数据
mlr.fit(trainingDS)

// 计算测试数据的预测结果
val predictions = mlr.predict(testingDS)
```
