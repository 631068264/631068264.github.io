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



# Flash Attention

[FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135)

![image-20230923114543837](https://cdn.jsdelivr.net/gh/631068264/img/202309231145944.png)

- SRAM的IO速度远大于GPU HBM的IO速度，在SRAM做运算搬运结果更快
- 将长度为N的句子的Q和{K，V}对分成诸多小块，外循环和内循环在长度轴N上进行，循环计算



**materialization**
“材料化”指的是将 N x N 的注意力矩阵存储或表示在内存中，特别是存储在GPU的高带宽内存（HBM）上的过程。通过平铺注意力矩阵并将其加载到片上SRAM（快速的片上内存）中，避免了在相对较慢的GPU HBM上完全材料化整个注意力矩阵。这种方法有助于提高FLASHATTENTION实现中注意力计算的效率和速度。



# Vision Transformer（VIT）

[An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale](https://arxiv.org/abs/2010.11929)

[ViT论文逐段精读【论文精读】](https://www.youtube.com/watch?v=FRFt3x0bO94&ab_channel=MuLi)

图像任务直接作用Transformer上面



把图片所有像素拉成一维数组放到Transformer，图片尺寸（224*224=507776），远远超过大模型可接受的序列长度。



VIT有足够规模的数据预训练效果比现有的残差网络效果接近甚至更好。除了抽取图像块和位置编码用了一些图像特有的这个归纳偏置，然后就可以使用和NLP一样的Transformer。

**和CNN相比，要少很多这种图像特有的归纳偏置。在中小数据集上的表现不如CNN。**

![image-20230927092717661](https://cdn.jsdelivr.net/gh/631068264/img/202309270927822.png)

## patch+位置信息

ViT的首要任务是将图转换成词的结构，将图片分割成小块，每个小块就相当于句子里的一个词。这里把每个小块称作Patch。

>假设：patch 16x16，224/16 = 14  序列长度=14x14=196

而**Patch Embedding**就是把每个Patch再经过一个全连接网络压缩成一定维度的向量。

> 假设：原图224x224x3(RGB channel) 转换成一个patch 的维度(16x16x3=768) 总共196个，通过一个全连接层（768x768），最终向量Patch(196x768) x 全连接层（768x768）= 向量(196x768) + cls_token(1x768) = embedding（197x168）

**加入cls_token**永远放在位置0，由于所有token之间都交换信息，其他的embedding表达的都是不同的patch的特征，而cls_token是要综合所有patch的信息，产生一个新的embedding，**来表达整个图的信息**。



**位置信息**

和向量维度都是一样的，直接和embedding（197x168）**相加**    = Embedded Patches(197x168)



图片尺寸改变会影响，patch size 和位置编码，使用更大尺寸图片微调有局限性。位置编码可以学习到2d信息。

![image-20230927092754838](https://cdn.jsdelivr.net/gh/631068264/img/202309270927384.png)

# CLIP

[Learning Transferable Visual Models From Natural Language Supervision](https://arxiv.org/abs/2103.00020)

[CLIP 论文逐段精读【论文精读】](https://www.youtube.com/watch?v=OZF1t_Hieq8&ab_channel=MuLi)

**真正把视觉和文字上语义联系到一起，做到zero shot推理，泛化性能远比有监督训练出来的模型厉害**

- 利用文本监督信号，而不是N选一这样的标签，训练集必须够大（WebImageText WIT）

- 给定图片去逐字逐句预测文本，而且对同一张图描述会很多，所以简化成衡量文字信息和图片是否配对，可以提高训练效率

  ![image-20230927153334437](https://cdn.jsdelivr.net/gh/631068264/img/202309271533472.png)

- 传统模型主要研究特征学习的能力，学习泛化性较好的特征，应用到下游任务时还需要微调，所以需要zero shot 迁移

## 预训练

<img src="https://cdn.jsdelivr.net/gh/631068264/img/202309271038087.png" alt="image-20230927103844040" style="zoom:50%;" />

通过各自的Encoder得到N个特征，CLIP通过这些特征进行[对比学习](https://lilianweng.github.io/posts/2021-05-31-contrastive/)（对角线是正样本，白色底是负样本）



```sh
# image_encoder - ResNet or Vision Transformer
# text_encoder - CBOW or Text Transformer
# I[n, h, w, c] - minibatch of aligned images  N个图像，height x wide x channel
# T[n, l] - minibatch of aligned texts  N个文本，l序列长度
# W_i[d_i, d_e] - learned proj of image to embed
# W_t[d_t, d_e] - learned proj of text to embed
# t - learned temperature parameter
# extract feature representations of each modality
I_f = image_encoder(I) #[n, d_i]
T_f = text_encoder(T) #[n, d_t]
# joint multimodal embedding [n, d_e]  单模态到多模态 ，归一化
I_e = l2_normalize(np.dot(I_f, W_i), axis=1)
T_e = l2_normalize(np.dot(T_f, W_t), axis=1)
# scaled pairwise cosine similarities [n, n]
logits = np.dot(I_e, T_e.T) * np.exp(t)
# symmetric loss function
labels = np.arange(n)
loss_i = cross_entropy_loss(logits, labels, axis=0)
loss_t = cross_entropy_loss(logits, labels, axis=1)
loss = (loss_i + loss_t)/2
```



学习的目标是学习这样一个嵌入空间，其中相似的样本对彼此靠近，而不相似的样本对相距很远。对比学习可以应用于有监督和无监督的环境。[在处理无监督数据时，对比学习是自监督学习](https://lilianweng.github.io/posts/2019-11-10-self-supervised/)中最强大的方法之一。



在对比学习的损失函数的早期版本中，仅涉及一个正样本和一个负样本。最近训练目标的趋势是在一批中包含多个正负对。





## 推理

![image-20230927105341647](https://cdn.jsdelivr.net/gh/631068264/img/202309271053700.png)

怎样推理分类

- 把所有的分类套入到prompt template，变成一个句子，然后和训练好的编码器进行encoder，得到N个文本特征。
- 将图片和训练好的编码器进行encoder，得到图片特征。和所有的文本特征，计算相似度
- 根据相似度可以得到分类

**真正使用的时候，object可以换成其他从来没有训练的单词，图片也可以随便，CLIP可以做到zeor shot，普通的分类模型固定类别N选一**



## 局限性

- 扩大CLIP规模无法扩大性能

- 不擅长细分类，抽象概念

- 图片和训练图集差得远一样表现差

- 只能从给定的类比判断是否类似，不能好像生成式那样生成新输出

- 数据利用率不高，需要太多数据

- 数据没有太多清洗，有数据偏见

- few shot效果反而不好

  ![image-20230927161037988](https://cdn.jsdelivr.net/gh/631068264/img/202309271610041.png)

# ViLT

[ViLT: Vision-and-Language Transformer Without Convolution or Region Supervision](https://arxiv.org/abs/2102.03334)



图像相关的模态，以前的模型通过区域特征的抽取，相当于目标检测任务，输出一些离散序列看出单词，输入到transformer和NLP模型做模态融合。



**但是模型在运行时间方面浪费好多。去掉卷积特征（预训练好的一个分类模型抽出来的特征图）和区域特征（根据特征图做的目标检测的那些框所带来的特征）带来的监督信号，加快运行时间。但是性能没有使用特征的强**  先预训练，再微调。

使用特征带来的坏处

- 运行效率不行。（抽出特征的时间比模态融合的时间长）
- 模型表达能力受限，因为预训练好的目标检测器，规模不大，用的数据集也小类别数不多，因为文本是没有限制的。

为什么用目标检测

- 需要语义强离散的特征表现形式，每个区域可以看出单词。
- 和下游任务有关（某个物体是否存在，物体在哪），和物体有强关联性

![image-20230929154514752](https://cdn.jsdelivr.net/gh/631068264/img/202309291545914.png)

## 改进

受到VIT启发。

- 使用patch emedding简化计算复杂度，保持性能
- 使用数据增强（image-text pair 对应的语义），尽量保证可以对应上
- NLP整个词mask调，（当时CV的完形填空的方法还没正式有）

衡量模型

- 图像和文本表达能力是否平衡（之前图像训练贵很多），参数量
- 模态的融合（做得不好，影响下游任务）

![image-20230929164228354](https://cdn.jsdelivr.net/gh/631068264/img/202309291642460.png)

对现有领域的信息，进行分类。加深对领域的理解。



## 模态融合

效果各有千秋

- single-stream approaches

  只用一个模型处理两个输入（图像和文本），直接向量串联，让transformer自己去学  （**使用了这个**）

- dual-stream approaches

  使用两个模型，对各自的输入，充分去挖掘单独模态里包含的信息，在某个时间点用transformer融合。引入更多参数，成本贵。

![image-20230929172310811](https://cdn.jsdelivr.net/gh/631068264/img/202309291723854.png)

```sh
I[n, h, w, c] - minibatch of aligned images  N个图像，height x wide x channel
T[n, l] - minibatch of aligned texts  N个文本，l序列长度

总序列长度= （N+L+2）x N
```

