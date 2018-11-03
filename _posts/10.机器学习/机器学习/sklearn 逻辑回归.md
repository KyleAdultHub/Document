---
title: sklearn 逻辑回归
date: "2018-11-3 23:06:00"
categories:
- 机器学习
- 机器学习
tags:
- sklearn
- 逻辑回归
toc: true
typora-root-url: ..\..\..
---

### 逻辑回归作用

可以做概率预测，也可用于分类，仅能用于线性问题。通过计算真实值与预测值的概率，然后变换成损失函数，求损失函数最小值来计算模型参数，从而得出模型。

### sklearn调用接口

```python
class sklearn.linear_model.LogisticRegression(penalty=’l2’, dual=False, tol=0.0001, C=1.0, fit_intercept=True, intercept_scaling=1, class_weight=None, *random_state=None*, solver=’liblinear‘, max_iter=100, multi_class=’ovr’, verbose=0, warm_start=False, n_jobs=1)
```

<!-- more -->

参数**

　　=> **penalty** : str, ‘l1’ or ‘l2’　　

　　　　LogisticRegression和LogisticRegressionCV默认就带了正则化项。penalty参数可选择的值为"l1"和"l2"，分别对应L1的正则化和L2的正则化，默认是L2的正则化。

　　　　在调参时如果我们主要的目的只是为了解决过拟合，一般penalty选择L2正则化就够了。但是如果选择L2正则化发现还是过拟合，即预测效果差的时候，就可以考虑L1正则化。

　　　　另外，如果模型的特征非常多，我们希望一些不重要的特征系数归零，从而让模型系数稀疏化的话，也可以使用L1正则化。

　　　　penalty参数的选择会影响我们损失函数优化算法的选择。即参数solver的选择，如果是L2正则化，那么4种可选的算法{‘newton-cg’, ‘lbfgs’, ‘liblinear’, ‘sag’}都可以选择。

　　　　但是如果penalty是L1正则化的话，就只能选择‘liblinear’了。这是因为L1正则化的损失函数不是连续可导的，而{‘newton-cg’, ‘lbfgs’,‘sag’}这三种优化算法时都需要损失函数的一阶或者二阶连续导数。而‘liblinear’并没有这个依赖。

　　=> **dual** : bool

　　　　对偶或者原始方法。Dual只适用于正则化相为l2 liblinear的情况，通常样本数大于特征数的情况下，默认为False

　　=> **tol** : float, optional

　　迭代终止判据的误差范围。

　　=> **C** : float, default: 1.0

　　　　C为正则化系数λ的倒数，通常默认为1。设置越小则对应越强的正则化。

　　=> **fit_intercept** : bool, default: True

　　　　是否存在截距，默认存在

　　=> **intercept_scaling** : float, default 1.

　　　　仅在正则化项为"liblinear"，且fit_intercept设置为True时有用。

　　=> **class_weight** : dict or ‘balanced’, default: None

　　　　class_weight参数用于标示分类模型中各种类型的权重，可以不输入，即不考虑权重，或者说所有类型的权重一样。如果选择输入的话，可以选择balanced让类库自己计算类型权重，

　　　　或者我们自己输入各个类型的权重，比如对于0,1的二元模型，我们可以定义class_weight={0:0.9, 1:0.1}，这样类型0的权重为90%，而类型1的权重为10%。

　　　　如果class_weight**选择****balanced**，那么类库会根据训练样本量来计算权重。某种类型样本量越多，则权重越低；样本量越少，则权重越高。

　　　　当class_weight为balanced时，类权重计算方法如下：n_samples / (n_classes * np.bincount(y))

　　　　n_samples为样本数，n_classes为类别数量，np.bincount(y)会输出每个类的样本数，例如y=[1,0,0,1,1],则np.bincount(y)=[2,3]  0,1分别出现2次和三次

　　　　**那么****class_weight****有什么作用呢？**

​        　　在分类模型中，我们经常会遇到两类问题：

​       　　 第一种是**误分类的代价很高**。比如对合法用户和非法用户进行分类，将非法用户分类为合法用户的代价很高，我们宁愿将合法用户分类为非法用户，这时可以人工再甄别，但是却不愿将非法用户分类为合法用户。这时，我们可以适当提高非法用户的权重。

​      　　 第二种是**样本是高度失衡的**，比如我们有合法用户和非法用户的二元样本数据10000条，里面合法用户有9995条，非法用户只有5条，如果我们不考虑权重，则我们可以将所有的测试集都预测为合法用户，这样预测准确率理论上有99.95%，但是却没有任何意义。

　　　　这时，我们可以选择balanced，让类库自动提高非法用户样本的权重。

　　=> **random_state** : int, RandomState instance or None, optional, default: None

　　　　随机数种子，默认为无，仅在正则化优化算法为sag,liblinear时有用。

　　=> **solver** : {‘newton-cg’, ‘lbfgs’, ‘liblinear’, ‘sag’, ‘saga’}

　　　　solver参数决定了我们对逻辑回归损失函数的优化方法，有4种算法可以选择，分别是：

　　　　　　a) liblinear：使用了开源的liblinear库实现，内部使用了坐标轴下降法来迭代优化损失函数。

　　　　　　b) lbfgs：拟牛顿法的一种，利用损失函数二阶导数矩阵即海森矩阵来迭代优化损失函数。

　　　　　　c) newton-cg：也是牛顿法家族的一种，利用损失函数二阶导数矩阵即海森矩阵来迭代优化损失函数。

　　　　　　d) sag：即随机平均梯度下降，是梯度下降法的变种，和普通梯度下降法的区别是每次迭代仅仅用一部分的样本来计算梯度，适合于样本数据多的时候，SAG是一种线性收敛算法，这个速度远比SGD快。

　　　　从上面的描述可以看出，newton-cg, lbfgs和sag这三种优化算法时都需要损失函数的一阶或者二阶连续导数，因此不能用于没有连续导数的L1正则化，只能用于L2正则化。而liblinear则既可以用L1正则化也可以用L2正则化。

　　　　同时，sag每次仅仅使用了部分样本进行梯度迭代，所以当样本量少的时候不要选择它，而如果样本量非常大，比如大于10万，sag是第一选择。但是sag不能用于L1正则化，所以当你有大量的样本，又需要L1正则化的话就要自己做取舍了

　　=> **max_iter** : int, optional

　　　　仅在正则化优化算法为newton-cg, sag and lbfgs 才有用，算法收敛的最大迭代次数。

　　=> **multi_class** : str, {‘ovr’, ‘multinomial’}, default: ‘ovr’　　　　

 　  　　OvR的思想很简单，无论你是多少元逻辑回归，我们都可以看做二元逻辑回归。具体做法是，对于第K类的分类决策，我们把所有第K类的样本作为正例，除了第K类样本以外的所有样本都作为负例，然后在上面做二元逻辑回归，得到第K类的分类模型。

　　　　其他类的分类模型获得以此类推。

​         　 而MvM则相对复杂，这里举MvM的特例one-vs-one(OvO)作讲解。如果模型有T类，我们每次在所有的T类样本里面选择两类样本出来，不妨记为T1类和T2类，把所有的输出为T1和T2的样本放在一起，把T1作为正例，T2作为负例，进行二元逻辑回归，

　　　   得到模型参数。我们一共需要T(T-1)/2次分类。

　　　   可以看出OvR相对简单，但分类效果相对略差（这里指大多数样本分布情况，某些样本分布下OvR可能更好）。而MvM分类相对精确，但是分类速度没有OvR快。如果选择了ovr，则4种损失函数的优化方法liblinear，newton-cg,lbfgs和sag都可以选择。

　　　   但是如果选择了multinomial,则只能选择newton-cg, lbfgs和sag了。

　　=> **verbose** : int, default: 0

　　=> **warm_start** : bool, default: False

　　=> **n_jobs** : int, default: 1

　　　　如果multi_class ='ovr'“，并行数等于CPU内核数量。当“solver”设置为“liblinear”时，无论是否指定“multi_class”，该参数将被忽略。如果给定值-1，则使用所有内核。

　　

 