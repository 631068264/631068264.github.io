---
layout:     post
rewards: false
title:   attention transformer原理 MOE
categories:
    - AI
tags:
   - 大模型



---



# self attention自注意力机制

可以考虑整个向量序列信息来计算，计算量大 [Attention Is All You Need 论文](https://arxiv.org/abs/1706.03762)

## 算法

https://www.youtube.com/watch?v=hYdO9CscNes&list=PLJV_el3uVTsMhtt7_Y6sgTHGHp1Vb2P2J&index=12&ab_channel=Hung-yiLee

- 每一个output **b**，都是**考虑所有的a 生成的**

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742145.png" alt="image-20230903100709678" style="zoom: 33%;" />




- 计算和a1最相关的向量，相关度由**alpha**表示，**计算alpha的方法有很多**

  向量$\vec{a}=[a_{1},a_{2},\cdot\cdot\cdot,a_{n}]$和向量${\vec{b}}=[b_{1},b_{2},\cdot\cdot,b_{n}]$ 內積定義

  $\vec{a}\cdot\vec{b}=\sum_{i=1}^{n}a_{i}b_{i}=a_{1}b_{1}+a_{2}b_{2}+\cdot\cdot\cdot+a_{n}b_{n}$
  
  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742208.png" alt="image-20230903101748213" style="zoom:33%;" />
  
  向量分别乘以对应不同的矩阵，得到q,k向量，再内积
  
- 通过关联性计算output b

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031051262.png" alt="image-20230903105128214" style="zoom: 33%;" />

  不一定是soft-max，做一个归一化处理。

  ![image-20230903105641949](https://cdn.jsdelivr.net/gh/631068264/img/202309031742467.png)

  b1 到b4是并行产生的

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742937.png" alt="image-20230903110158141" style="zoom:33%;" />

  

## 矩阵算法

  

  可以把所有向量a 看成是一个矩阵I，得到QKV矩阵

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742140.png" alt="image-20230903124447499" style="zoom:33%;" />

  获取alpha

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742740.png" alt="image-20230903124937885" style="zoom:33%;" />

  获取ouput

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031256610.png" alt="image-20230903125639542" style="zoom:33%;" />

  模型需要训练QKV

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742065.png" alt="image-20230903125852780" style="zoom:33%;" />



## multi-head

一个q代表一种相关性，需要多种就要多个q，超参数



q 分别乘以不同矩阵得到不同的q，独立计算就行和one head一样

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742688.png" alt="image-20230903130428827" style="zoom:33%;" />

等到output

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742244.png" alt="image-20230903130819618" style="zoom:33%;" />

## 位置信息

self-attention没有位置信息，可以用

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742271.png" alt="image-20230903131310074" style="zoom:33%;" />

## 与CNN RNN对比

### CNN

self-attention是更复杂的CNN

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742142.png" alt="image-20230903132429038" style="zoom:33%;" />

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742678.png" alt="image-20230903132615979" style="zoom:33%;" />

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031327608.png" alt="image-20230903132702489" style="zoom:33%;" />

### RNN

![image-20230903133031247](https://cdn.jsdelivr.net/gh/631068264/img/202309031330295.png)

主要两个不同

- 单向RNN只能考虑最先输入的向量，而不是整个句子。双向RNN，最右边的黄色output，很容易忘记最左边的蓝色input。但是self-attention可以通过QK联系起来。
- self-attention可以并行计算，RNN只能等上一个向量处理完后的输出才作为下一个的输入，不能并行计算。

### 图GNN

图的节点也可以当成向量，计算alpha矩阵时候可以只计算有边连接的节点

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742362.png" alt="image-20230903134114337" style="zoom:33%;" />

# Transformer

seq2seq，output的长度由model决定。通常有两步份组成。 encoder , decoder

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309031742622.png" alt="image-20230903162028389" style="zoom:25%;" />

## Encoder 原始结构

x是输入，h是输出，每个block，由不同layer构成。

![image-20230903162636479](https://cdn.jsdelivr.net/gh/631068264/img/202309031742719.png)

Transformer使用不同的方法，**block详解**。使用**residual(input + output 作为下一层input) + layer norm**



FC是Fully Connected Neural Network 全连接神经网络 也叫前馈神经网络（Feedforward Neural Network，FFN）

![image-20230903164306698](https://cdn.jsdelivr.net/gh/631068264/img/202309031742702.png)



完整Transformer encode, 使用 block 会重复N遍

![image-20230903164501385](https://cdn.jsdelivr.net/gh/631068264/img/202309031742308.png)



## Decoder

### autoregressive

#### 先忽略encoder input

在encoder输出里面加一个BOS(begin special token) ， 输出常见字表，取softamx后的最大值的那个字。

![image-20230903171218074](https://cdn.jsdelivr.net/gh/631068264/img/202309031742965.png)

这个字作为下一个输入

![image-20230903171556705](https://cdn.jsdelivr.net/gh/631068264/img/202309031742464.png)

decoder大概结构，与encoder非常相似，**但是decoder 多了一个Masked multi-head self-attention**

![image-20230903171755885](https://cdn.jsdelivr.net/gh/631068264/img/202309031742692.png)

#### Masked multi-head self-attention

一般的self-attention生成b，要考虑整个seq的信息。

但是masked 只考虑比它前的向量信息(生成b3，只考虑a1~a3)

![image-20230903172305885](https://cdn.jsdelivr.net/gh/631068264/img/202309031742316.png)

Why  Masked : 因为decoder的input决定这样，输入也是一个个进来，不可能考虑到后面的向量。

#### decoder 什么时候停

output seq 长度由model自己决定。 在vocabulary 加入stop token , 和begin token一样不一样都可以。

![image-20230903173445138](https://cdn.jsdelivr.net/gh/631068264/img/202309031742667.png)

![image-20230903173813726](https://cdn.jsdelivr.net/gh/631068264/img/202309031738783.png)

### Non autoregressive

![image-20230903191908611](https://cdn.jsdelivr.net/gh/631068264/img/202309031919675.png)

NAT input一堆begin，output整句输出。

when stop，（output 长度）

- 需要另外一个predictor预测
- 直接输出一个很长的output，出现stop token就忽略的output token

优点：

- 生成速度快比AT，可以output可以并行生成
- 可以很好控制输出长度

缺点

- 没有AT的表现好（why [multi-modality](https://youtu.be/jvyKmU4OM3c)）

## Transformer的endcoder和decoder结合 cross attention

![image-20230903193622281](https://cdn.jsdelivr.net/gh/631068264/img/202309031936325.png)

cross attention 做了什么。用endoder的output和mask的output，计算出来的V作为下一层（FC层）的input

![image-20230903194443259](https://cdn.jsdelivr.net/gh/631068264/img/202309031944333.png)

## train loss

每次decoder的output，都是对vocabulary(词汇表)的一次分类问题

每一个output，都要和其正确答案（vocabulary one-hot vector）进行cross entropy ，所有的cross entropy总和最小。

![image-20230903195641484](https://cdn.jsdelivr.net/gh/631068264/img/202309031956541.png)

训练时候，decoder给正确答案。

![image-20230903200349798](https://cdn.jsdelivr.net/gh/631068264/img/202309032003855.png)

## tips

- [Copy Mechanism](https://www.youtube.com/watch?v=VdOyqNQ9aww&ab_channel=Hung-yiLee) 从input复制某些词汇到output 场景： 翻译，chat，summary  

- Guided Attention  强迫Attention有固定模式，Attention不能漏，语序不能错  场景：语音识别、TTS  Monotonic Attention ，Location-aware attention

- Beam Search  decoder的output，不一定选最大可能那个，帮助寻找较好路径 （对需要创造性的任务效果不好，有确切答案的任务效果好）

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309032020028.png" alt="image-20230903202030966" style="zoom:25%;" />

- Scheduled Sampling 训练测试要加入一些错误的列子，效果会更好







# MOE

[Outrageously Large Neural Networks: The Sparsely-Gated Mixture-of-Experts Layer](https://arxiv.org/abs/1701.06538)

稀疏门控专家混合模型 ( Sparsely-Gated MoE):旨在**实现条件计算**，即神经网络的某些部分以每个样本为基础进行激活，作为一种显著增加模型容量和能力而不必成比例增加计算量的方法。

![image-20230911210451264](https://cdn.jsdelivr.net/gh/631068264/img/202309112104424.png)

为了保证稀疏性和均衡性（为了不让某个expert专家权重特别大），对softmax做了如下处理 :

- 引入**KeepTopk**，这是个离散函数，将top-k之外的值强制设为负无穷大，从而softmax后的值为0。（合理听取某些专家建议，其他就不学习）
- 加noise，这个的目的是为了做均衡，这里引入了一个Wnoise的参数，后面还会在损失函数层面进行改动。

![image-20230911211622412](https://cdn.jsdelivr.net/gh/631068264/img/202309112116468.png)

将大模型拆分成多个小模型（**每个小模型就是一个专家**），对于一个样本来说，无需经过所有的小模型去计算，而**只是激活一部分小模型进行计算这样就节省了计算资源**。稀疏门控 MOE，实现了模型容量超过1000倍的改进，并目在现代 GPU 集群的计算效率损失很小
