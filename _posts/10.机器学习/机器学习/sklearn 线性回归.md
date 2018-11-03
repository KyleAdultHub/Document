---
title: sklearn 线型回归
date: "2018-10-31 23:03:00"
categories:
- 机器学习
- 机器学习
tags:
- sklearn
- 线型回归
toc: true
typora-root-url: ..\..\..
---

## 简单线性回归

线性回归是数据挖掘中的基础算法之一，从某种意义上来说，在学习函数的时候已经开始接触线性回归了，只不过那时候并没有涉及到误差项。线性回归的思想其实就是解一组方程，得到回归函数，不过在出现误差项之后，方程的解法就存在了改变，一般使用最小二乘法进行计算。

<!-- more -->

### 使用sklearn.linear_model.LinearRegression进行线性回归

sklearn对Data Mining的各类算法已经有了较好的封装，基本可以使用`fit`、`predict`、`score`来训练、评价模型，并使用模型进行预测，一个简单的例子如下：

```python
from sklearn import linear_model
clf = linear_model.LinearRegression()
X = [[0,0],[1,1],[2,2]]
y = [0,1,2]
clf.fit(X,y)
print(clf.coef_)
[ 0.5 0.5]
print(clf.intercept_)
1.11022302463e-16
```

`LinearRegression`已经实现了多元线性回归模型，当然，也可以用来计算一元线性模型，通过使用list[list]传递数据就行。下面是`LinearRegression`的具体说明。

#### 使用方法

##### 实例化

sklearn一直秉承着简洁为美得思想设计着估计器，实例化的方式很简单，使用`clf = LinearRegression()`就可以完成，但是仍然推荐看一下几个可能会用到的参数：

- `fit_intercept`：是否存在截距，默认存在
- `normalize`：标准化开关，默认关闭

还有一些参数感觉不是太有用，就不再说明了，可以去官网文档中查看。

##### 回归

其实在上面的例子中已经使用了`fit`进行回归计算了，使用的方法也是相当的简单。

- `fit(X,y,sample_weight=None)`：`X`,`y`以矩阵的方式传入，而`sample_weight`则是每条测试数据的权重，同样以`array`格式传入。
- `predict(X)`：预测方法，将返回预测值`y_pred`
- `score(X,y,sample_weight=None)`：评分函数，将返回一个小于1的得分，可能会小于0

##### 方程

`LinearRegression`将方程分为两个部分存放，`coef_`存放回归系数，`intercept_`则存放截距，因此要查看方程，就是查看这两个变量的取值。

#### 多项式回归

其实，多项式就是多元回归的一个变种，只不过是原来需要传入的是X向量，而多项式则只要一个x值就行。通过将x扩展为指定阶数的向量，就可以使用`LinearRegression`进行回归了。sklearn已经提供了扩展的方法——`sklearn.preprocessing.PolynomialFeatures`。利用这个类可以轻松的将x扩展为X向量，下面是它的使用方法：

```
>>> from sklearn.preprocessing import PolynomialFeatures
>>> X_train = [[1],[2],[3],[4]]
>>> quadratic_featurizer = PolynomialFeatures(degree=2)
>>> X_train_quadratic = quadratic_featurizer.fit_transform(X_train)
>>> print(X_train_quadratic)
[[ 1  1  1]
 [ 1  2  4]
 [ 1  3  9]
 [ 1  4 16]]
```

经过以上处理，就可以使用`LinearRegression`进行回归计算了。