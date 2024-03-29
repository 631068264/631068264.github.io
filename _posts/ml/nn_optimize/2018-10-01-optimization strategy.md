---
layout:     post
rewards: false
title:      NN优化策略
categories:
    - ml
tags:
    - nn优化
---

# 正交化方法（Orthogonalization）

**每次只调试一个参数，保持其它参数不变**，而得到的模型某一性能改变是一种最常用的调参策略。
Orthogonalization的核心在于**每次调试一个参数只会影响模型的某一个性能**。

对应到机器学习监督式学习模型中，可以大致分成四个独立的“功能”：
- Fit training set well on cost function
- Fit dev set well on cost function
- Fit test set well on cost function
- Performs well in real world

early stopping在模型功能调试中并不推荐使用。因为early stopping在提升验证集性能的同时降低了训练集的性能。
也就是说early stopping同时影响两个“功能”，不具有独立性、正交性。

# 单一数字评估指标（Single number evaluation metric）
构建、优化机器学习模型时，**单值评价指标非常必要**。有了量化的单值评价指标后，我们就能根据这一指标比较不同超参数对应的模型的优劣，从而选择最优的那个模型。
例如[F1 socre](ml/2018/05/10/分类-指标trick/#f-score) or 平均错误率

# 满足和优化指标（Satisficing and optimizing metrics）
有时候，要把所有的性能指标都综合在一起，构成单值评价指标是比较困难的。

解决办法是，我们可以把**某些性能作为优化指标**（Optimizing metic），**寻求最优化值**；
而某些性能作为**满意指标**（Satisficing metic），只要**满足阈值**就行了。

# 训练/开发/测试集划分（Train/dev/test distributions）
Train/dev/test sets如何设置对机器学习的模型训练非常重要，合理设置能够大大提高模型训练效率和模型质量。
应该尽量保证dev sets和test sets来源于**同一分布**且都反映了实际样本的情况。

当样本数量不多（小于一万）的时候，通常将Train/dev/test sets的比例设为60%/20%/20%，
在没有dev sets的情况下，Train/test sets的比例设为70%/30%。当样本数量很大（百万级别）的时候，
通常将相应的比例设为98%/1%/1%或者99%/1%。


对于dev sets数量的设置，应该遵循的准则是通过dev sets能够检测不同算法或模型的区别，以便选择出更好的模型。
对于test sets数量的设置，应该遵循的准则是通过test sets能够反映出模型在实际中的表现。
实际应用中，可能只有train/dev sets，而没有test sets。这种情况也是允许的，只要算法模型没有对dev sets过拟合。但是，条件允许的话，最好是有test sets，实现无偏估计。

# 什么时候该改变开发/测试集和指标？（When to change dev/test sets and metrics）
算法模型的**评价标准**有时候需要根据**实际情况进行动态调整**，不同的目的，不同的指标，不同的model。
动态改变评价标准的情况是dev/test sets与实际使用的样本分布不一致。比如猫类识别样本图像分辨率差异

# 对比 human-level performance & ML model performance
通过比较找出优化方向，实现优化。
通常，我们把training error与human-level error之间的差值称为bias，
也称作avoidable bias；把dev error与training error之间的差值称为variance。
根据bias和variance值的相对大小，可以知道算法模型是否发生了欠拟合或者过拟合。

例如猫类识别的例子中，如果human-level error为1%，training error为8%，dev error为10%。
由于training error与human-level error相差7%，dev error与training error只相差2%，
所以目标是尽量在训练过程中减小training error，即减小偏差bias。如果图片很模糊，肉眼也看不太清，human-level error提高到7.5%。
这时，由于training error与human-level error只相差0.5%，dev error与training error只相差2%，
所以目标是尽量在训练过程中减小dev error，即方差variance。这是相对而言的。

human-level performance定义较难，不同人可能选择的human-level performance基准是不同的，
选择什么样的human-level error，有时候会影响bias和variance值的相对变化。**一般来说，我们将表现最好的那一组**。

解决avoidable bias的常用方法包括：
- Train bigger model
- Train longer/better optimization algorithms: momentum, RMSprop, Adam
- NN architecture/hyperparameters search

解决variance的常用方法包括：
- More data
- Regularization: L2, dropout, data augmentation
- NN architecture/hyperparameters search

# 误差分析（Carrying out error analysis）
error analysis可以同时评估多个影响模型性能的因素，通过各自在**错误样本中所占的比例**来判断其重要性。
比例越大，影响越大，越应该花费时间和精力着重解决这一问题。这种error analysis让我们改进模型更加有针对性，从而提高效率。
例如假阳性（false positives）和假阴性（false negatives）

监督式学习中，训练样本有时候会出现输出y标注错误的情况，即incorrectly labeled examples。
利用上节内容介绍的error analysis，统计dev sets中所有分类错误的样本中**incorrectly labeled data所占的比例**。
根据该比例的大小，决定是否需要修正所有incorrectly labeled data，还是可以忽略。

- 如果你打算修正开发集上的部分数据，那么最好也对测试集做**同样的修正**以确保它们继续来自**相同的分布**。
- 同时检验算法判断正确和判断错误的样本，要检查算法出错的样本很容易，只需要看看那些样本是否需要修正，
但还有可能有些样本算法判断正确，那些也需要修正。因为算法有可能**因为运气好把某个东西判断**对了，
通常判断错的次数比判断正确的次数要少得多，检查正确的上要花的时间长得多，通常不这么做，但也是要考虑到的。
- **修正训练集中的标签其实相对没那么重要**，你可能决定**只修正开发集和测试集中的标签**，因为它们通常比训练集小得多，
你可能不想把所有额外的精力投入到修正大得多的训练集中的标签，所以这样其实是可以的。**开发集和测试集来自同一分布非常重要**，
但如果你的**训练集来自稍微不同的分布**，通常这是一件很合理的事情。


# 使用来自不同分布的数据，进行训练和测试（Training and testing on different distributions）
面对train set与dev/test set分布不同的情况，有两种解决方法
- 将train set和dev/test set完全混合，然后在随机选择一部分作为train set，另一部分作为dev/test set。
优点是实现train set和dev/test set分布一致。缺点dev/test set大部分来自非目标的分布（train set）,**达不到验证效果,不建议使用**。
- 将原来的train set和一部分dev/test set组合当成train set，剩下的dev/test set分别作为dev set和test set。
dev/test set全部来自dev/test set，**保证了验证集最接近实际应用场合。这种方法较为常用，而且性能表现比较好**。

# 数据分布不匹配时，偏差与方差的分析（Bias and Variance with mismatched data distributions）
当你的训练集来自和开发集、测试集不同分布时，分析偏差和方差的方式可能不一样。
差值的出现可能来自算法本身，也可能来自于样本分布不同。因此不能简单认为出现了Variance。

从原来的train set中分割出一部分作为train-dev set，train-dev set不作为训练模型使用，而是与dev set一样用于验证。
就有training error、training-dev error和dev error三种error，其中，training error与training-dev error的差值
反映了variance；training-dev error与dev error的差值反映了data mismatch problem，即样本分布不一致。

总结一下human-level error、training error、training-dev error、dev error以及test error之间的差值关系和反映的问题：
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNc79gy1fvt2kxt89hj31j60s2gmz.jpg)

# 处理数据不匹配问题（Addressing data mismatch）
您的训练集来自和开发测试集不同的分布，如果错误分析显示你有一个数据不匹配的问题该怎么办。

- Carry out manual error analysis to try to understand difference between training dev/test sets
- Make training data more similar; or collect more data similar to dev/test sets
**尝试弄清楚开发集和训练集到底有什么不同**，当你了解开发集误差的性质时，
你就知道，开发集有可能跟训练集不同或者更难识别，那么**你可以尝试把训练数据变得更像开发集一点**。或者看看是否有办法收集更多看起来像开发集的数据作训练。

为了让train set与dev/test set类似，我们可以使用人工数据合成的方法（artificial data synthesis），但你的**学习算法可能会对合成的这一个小子集过拟合**。

# 面对全新ML应用
尽快建立你的第一个简单模型，然后快速迭代。
它可以是一个快速和粗糙的实现（quick and dirty implementation）初始系统的全部意义在于，
有一个学习过的系统，有一个训练过的系统，让你确定偏差方差的范围，
就可以**知道下一步应该优先做什**么，让你能够进行错误分析，可以**观察一些错误，然后想出所有能走的方向，哪些是实际上最有希望的方向**。

# 迁移学习（Transfer learning）
深度学习非常强大的一个功能之一就是有时候你可以将已经训练好的模型的一部分知识（网络结构）直接应用到另一个类似模型中去。

迁移学习，重新训练权重系数，如果需要构建新模型的样本数量较少，那么可以像刚才所说的，
只训练输出层的权重系数$W^{[L]},\ b^{[L]}$，保持其它层所有的权重系数$W^{[l]},\ b^{[l]}$不变。
这种做法相对来说比较简单。如果样本数量足够多，那么也可以只保留网络结构，重新训练所有层的权重系数。这种做法使得模型更加精确，
因为毕竟样本对模型的影响最大。**选择哪种方法通常由数据量决定**。

顺便提一下，如果重新训练所有权重系数，初始$W^{[l]},\ b^{[l]}$由之前的模型训练得到，
这一过程称为**pre-training**。之后，不断调试、优化$W^{[l]},\ b^{[l]}$的过程称为fine-tuning

迁移学习之所以能这么做的原因是，神经网络浅层部分能够检测出许多图片固有特征，第一个训练好的神经网络已经帮我们实现如何提取图片有用特征了。 
因此，即便是即将训练的第二个神经网络样本数目少，仍然可以根据第一个神经网络结构和权重系数得到健壮性好的模型。
迁移学习可以保留原神经网络的一部分，再添加新的网络层。具体问题，具体分析，可以去掉输出层后再增加额外一些神经层。

应用场合
- Task A and B have the same input x.
- You have a lot more data for Task A than Task B.
- Low level features from A could be helpful for learning B.

# 多任务学习（Multi-task learning）
多任务学习是使用单个神经网络模型来实现多个任务。多任务学习类似将多个神经网络融合在一起，用一个网络模型来实现多种分类效果。

值得一提的是，Multi-task learning与Softmax regression的区别在于Softmax regression是single label的，
即输出向量y只有一个元素为1；而Multi-task learning是multiple labels的，即输出向量y可以有多个元素为1。

应用场合
- Training on a set of tasks that could benefit from having shared lower-level features.
- Usually: Amount of data you have for each task is quite similar.
- Can train a big enough neural network to do well on all the tasks.

# 端到端的深度学习（end-to-end deep learning）
以前有一些数据处理系统或者学习系统，它们需要多个阶段的处理。那么端到端深度学习就是忽略所有这些不同的阶段，用单个神经网络代替它。

端到端深度学习做的是，你训练一个巨大的神经网络，直接学到了和之间的函数映射，直接绕过了其中很多步骤。如果训练样本足够大，
神经网络模型足够复杂，那么end-to-end模型性能比传统机器学习分块模型更好。
实际上，end-to-end让神经网络模型内部去自我训练模型特征，自我调节，增加了模型整体契合度。
优点：
- Let the data speak
- Less hand-designing of components needed
缺点：
- May need large amount of data
- Excludes potentially useful hand-designed

                                                  





