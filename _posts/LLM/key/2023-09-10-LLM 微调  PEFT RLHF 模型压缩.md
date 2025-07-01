---
layout:     post
rewards: false
title:   LLM 微调  PEFT RLHF 模型压缩
categories:
    - AI
tags:
   - 大模型



---

# 微调fintune

https://www.bilibili.com/list/watchlater?bvid=BV1Rz4y1T7wz&oid=575237949&p=4

## why

fintune作用

- 引导模型获得更一致的输出
- 减少幻觉
- 根据特定用例定制模型
- 过程与模型早期的训练类似



Prompting和Finetuning对比

|      | Prompting 提示词                                             | Finetuning                                                   |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 优点 | 没有数据可以开始<br/>前期成本较小<br />无需技术知识<br />通过检索等方法让更多数据，有选择性进入提示词 (RAG) | 几乎无限的数据适合<br />了解新信息 ，更正不正确的信息<br />如果模型较小，则后续成本较低<br />RAG |
| 缺点 | 适合的数据少得多<br/>上下文太长，忘记数据<br />幻觉<br />RAG 丢失或获取不正确的数据<br /> | 需要优质数据<br />前期计算成本（算力，数据准备成本）<br />需要一些技术知识，尤其是。数据了解准备等 |
| 范围 | 通用模型，项目                                               | 垂直领域，企业，生产私有化部署，隐私保护                     |

微调好处

|        |                                                    |
| ------ | -------------------------------------------------- |
| 表现   | 停止幻觉<br />提高一致性<br />减少不需要的信息     |
| 隐私   | 本地部署，防止泄漏，无违规行为                     |
| 成本   | 每个请求的成本更低<br />提高透明度<br />更好的控制 |
| 可靠性 | 控制正常运行时间<br />更低的延迟<br />             |

指令微调（instruction finetuning）泛化

- 可以使用模型预存数据，可以推广到其他数据，可不是仅限于微调数据集中。

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309121026365.png" alt="image-20230912102617270"  />

  ![image-20230912210506274](https://cdn.jsdelivr.net/gh/631068264/img/202309122105315.png)
  
  

## 数据准备

准备大约500~1k条

Q&A

- prompt template（同一个instruction,不同表达，泛化能力更强）

  ![image-20230912103242726](https://cdn.jsdelivr.net/gh/631068264/img/202309121032815.png)

- 使用其他LLM(chatgpt )等参考 [Stanford Alpaca](https://github.com/tatsu-lab/stanford_alpaca)

数据质量

- 更高品质
- 相同语义输入输出多样性
- 真实的
- 更多的

准备数据的步骤

https://www.bilibili.com/list/watchlater?bvid=BV1Rz4y1T7wz&oid=575237949&p=5

- 收集指令-响应对，或者使用相应提示语模板

- Tokenizing(编码解码成数组)：填充、截断

  padding

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309021211420.png" alt="image-20230902121105050" style="zoom:33%;" />

  truncation 截尾

  <img src="https://cdn.jsdelivr.net/gh/631068264/img/202309021212205.png" alt="image-20230902121224131" style="zoom:33%;" />

- 拆分数据集：训练/测试





## 训练

- 添加训练数据
- 通过模型反向传播
- 更新权重

超参数

- 学习率
- 学习率调度器
- 优化超参数





## 灾难性遗忘Catastrophic forgetting

”灾难性溃忘"发生是因为全面微调过程会改变原始LLM的权重。

- 可以显著提高在特定任务上的性能，但是减低其他任务的性能



How to avoid

- 确认灾难性遗忘是否影响到你使用场景 Confirm if catastrophic forgetting affects your use case.
- 如果需要其他任务的泛化能力，可以多任务一齐微调。良好的多任务微调，**可能需要50-100,000个例子跨多个任务** If you need the generalization ability for other tasks, you can fine-tune multiple tasks together.
- 使用PEFT保留原始LLM权重，并且只训练少量特定任务的适配器层和参数。由于大部分预训练权重保持不变，PEFT对灾难性遗忘有更大的鲁棒性





## 评估

- 人工评价
- 良好的测试数据至关重要
    - 高质量
    - 准确的
    - 广义的
    - 训练数据中未见



- 误差分析
    - 在微调之前了解基本模型行为
    - 对错误进行分类：迭代数据以修复这些问题

- 微调任务与模型大小

    - 复杂性：输出token越多越难

        - 摘录：“阅读”更容易

        - 扩展：“写作”更难

    - 任务组合比一项难

      一次或一步完成几件事

### 具体方法

ROUGE (Recall-Oriented Understudy for Gisting Evaluation) is primarily used for comparing automatically generated summaries with manually created reference summaries.

ROUGE (召回率导向的摘要评估)主要用于通过将自动生成的摘要与人工生成的参考摘要进行比较，来评估摘要的质量。

BLEU (双语评估) 用于翻译评估

![image-20230912220421353](https://cdn.jsdelivr.net/gh/631068264/img/202309122204402.png)

unigram 一个词，bigram 两个词，n-gram n个词



#### ROUGE

第一种评估方法，**没有考虑单词顺序，随便改个词可能改变句意，但是依然高分**，增大n可以改善问题
![image-20230913202242568](https://cdn.jsdelivr.net/gh/631068264/img/202309132022692.png)

寻找在生成的输出和参考句子中都存在的**最长公共子序列的长度** LCS Longest common subsequence ，**来确认n**。同一任务，比较分数才有用。

![image-20230913203838540](https://cdn.jsdelivr.net/gh/631068264/img/202309132038650.png)

![image-20230913204017235](https://cdn.jsdelivr.net/gh/631068264/img/202309132040283.png)

#### BLEU

BLEU metric = Avg(precision across range of n-gram sizes)

![image-20230913205435559](https://cdn.jsdelivr.net/gh/631068264/img/202309132054610.png)

## 更好更复杂的标准

不同模型之间比较，[排行榜](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard)

- ARC 是一套小学问题。
- HellaSwag 是对常识的测试。
- MMLU 是一个涵盖基础数学的多任务指标，美国历史、计算机科学、法律等。
- TruthfulQA 衡量模型重现网上常见谎言的倾向。

![image-20230913211415461](https://cdn.jsdelivr.net/gh/631068264/img/202309132114509.png)



# SFT  Supervised fine-tuning

有监督微调使用有标签的数据( Label Data )来调整已经预练的 LLMs，使其更适应某一特定场景任务。



指令微调 Instruction Tuning，特殊之处在于其数据集的结构 **<指令，输出> 对**，由人类指令和期望的输出组成进行配对。这种数据结构使得指令微调专注于让模型理解和遵循人类指令。作为有监督微调的一种特殊形式，专注于通过理解和遵循人类指今来增强大型语言模型的能力和可控性.





# PEFT（Parameter-Efficient Fine-Tuning）

- [Parameter-Efficient Fine-Tuning (PEFT) for LLMs: A Comprehensive Introduction](https://towardsdatascience.com/parameter-efficient-fine-tuning-peft-for-llms-a-comprehensive-introduction-e52d03117f95)

- [大型语言模型与生成式AI——参数高效微调1——参数高效微调（PEFT）](https://www.youtube.com/watch?v=hsDaw4S5GZY&list=PLiuLMb-dLdWL4KBaU3FTM5f_oMcSvXcZw&index=18)



## PEFT 方法概览

PEFT技术都对transformer block或self-attention做出了变化

![img](https://cdn.jsdelivr.net/gh/631068264/img/202309091028590.png)

## Additive Methods

增量方法的目标是添加一组额外的参数或网络层来增强模型。在微调数据时，您只更新这些新添加参数的权重。这样可以使训练计算上更容易，并且适应较小的数据集（初步考虑在100-500左右，上限约为100,000）。

### Adapters

在Transformer子层之后添加小的FC层，并学习这些参数。The goal of adapters is to add small fully connected networks after Transformer sub-layers and learn those parameters.

伪代码例子

```python
def transformer_block_adapter(x):
    """Pseudo code from [2] """
    residual = x
    x = self_attention(x)
    x = FFN(x)  # adapter
    x = layer_norm(x + residual)
    residual = x
    x = FFN(x)
    x = FFN(x)  # adapter
    x = layer_norm(x + residual)
    return x
```

## Soft-Prompts

相对于hard-prompts，在训练集中穷举对于同一个问题不同表达方式，Soft-Prompts就是要避免创建这样的训练集。

核心思想是基础模型不是优化文本本身，而是优化提示文本的连续表示（例如一种可学习的张量形式）。这可以是一种嵌入（embedding）的形式，或者是应用于该嵌入的某种转换。随着我们的讨论深入，这些技术将会更详细地探讨。

**可以理解成看模型对语义的理解**，而不是穷举或者优化文字。

hard-prompts，代表自然语言的 Token 在某种意义上是硬的，因为**它们每一个都对应于嵌入向量空间中的一个固定位置。**

Soft-Prompts ，软提示并不是自然语言的固定离散词语，你可以把它们看作是可以在**连续的多维嵌入空间中取任何值的虚拟 Token。**

通过监督学习，模型学习了这些虚拟Token 的值，以最大化给定任务的性能

### Prompt-Tuning 提示词微调

和[提示工程  Prompt-engineer](https://www.promptingguide.ai/zh)不一样

在提示词工程里面，你需要修改你的提示词内容，以得到你想要的。你可以通过尝试不同的词汇或短语，甚至更复杂的方式，如**包括一次或少次推理的示例（One-shot or Few-shot Inference）**

缺点：

- 需要大量的手动努力来编写和尝试不同的提示。
- 受到上下文窗口长度的限制，而且到头来你可能仍然无法达到你的任务所需的性能。

![image-20230914215854994](https://cdn.jsdelivr.net/gh/631068264/img/202309142158051.png)

- 这组可训练的 Token 被称为软提示 (Soft Prompt）它被添加到代表你的输入文本的嵌入向量之前。
- 软提示向量的长度与语言 Token 的嵌入向量的长度相同。
- 20~100个虚拟token就足以获得良好的性能。
- **随着时间的推移，软提示的嵌入向量会被更新，以优化模型的提示完成。**

![img](https://cdn.jsdelivr.net/gh/631068264/img/202309091141851.png)

通过使用软提示，我们的目标是向基础模型中添加与当前任务更具体的信息。通过prompt tuning，我们通过在网络开始处创建一组prompt token的参数来实现这一目标。

To find a representation of the soft prompt, we create a separate set of embeddings for the static prompt used during training. We concatenate the output embeddings with the sequence embeddings. We use this new information to pass into the language model. Creating this dual information allows us to learn a parameterization of the soft prompt without needing to create many prompts for the same task.



```python
def prompt_tuning(seq_tokens, prompt_tokens):
    """ Pseudo code from [2]. """
    x = seq_embedding(seq_tokens)
    soft_prompt = prompt_embedding(prompt_tokens)
    model_input = concat([soft_prompt, x], dim=seq)
    return model(model_input)
```

### Prefix Tuning

Prefix tuning和prompt tuning的区别在于，输入会喂给FC，而prompt tuning只是embedding，此外，我们还通过完全连接的网络学习了用于prefix tuning的软提示的附加参数。在训练完全连接网络后，我们会将其丢弃，只使用软提示作为输入。

 fed to all layers of the transformer , use the soft-prompt as input for  a fully connected network,

```python
def transformer_block_prefix_tuning(x, soft_prompt):
    """ Pseudo code from [2] """
    soft_prompt = FFN(soft_prompt)
    model_input = concat([soft_prompt, x], dim=seq)
    return model(model_input)
```

![image-20231128223731366](https://cdn.jsdelivr.net/gh/631068264/img/202311282237800.png)

![image-20231128223756605](https://cdn.jsdelivr.net/gh/631068264/img/202311282243993.png)

### P-Tuning

![img](https://cdn.jsdelivr.net/gh/631068264/img/202309091249964.png)

可以将P-Tuning视为使用LSTM对提示进行编码的prompt tuning。Encoding the prompt using an LSTM.



P-Tuning旨在解决作者们注意到的两个问题。

- 首先是传递给模型的词嵌入的离散性。作者们认为，如果嵌入是随机初始化的，然后通过随机梯度下降进行优化，模型很可能陷入局部最小值。

- 第二个问题是词嵌入的关联性。在prompt-tuning和prefix-tuning的参数化中，软提示在技术上是相互独立的。作者们希望采用一种方法使提示令牌彼此依赖。

P-Tuning sets out to solve two problems the authors noticed. 

- The first is the discreteness of the word embeddings passed to the model. The authors argue that if the embeddings are randomly initialized and then optimized through Stochastic Gradient Descent, the model is likely to fall into a local minima.
- They second is association of the word embeddings. With the parameterization in prompt-tuning and prefix-tuning, the soft prompts are technically independent of each other. The authors wanted an approach that made the prompt tokens dependent on each other.

The example sequence “The capital of Britain is [MASK]”. Here the prompt is “The capital of … is …”, the context is “Britain” and the target is [MASK].

We can use this formulation to create two sequences of tokens, everything before the context and everything after the context before the target.


```python
def p_tuning(seq_tokens, prompt_tokens):
    """Pseudo code for p-tuning created by Author."""
    h = prompt_embedding(prompt_tokens)
    h = LSTM(h, bidirectional=True)
    h = FFN(h)

    x = seq_embedding(seq_tokens)
    model_input = concat([h, x], dim=seq)

    return model(model_input)
```

是一种**基于参数微调**的方法。在P-Tune中，微调的目标是调整模型中的参数，使其能够更好地适应新的任务或数据集。与LORA不同的是，P-Tune更加直接，仅仅调整模型中的参数，而不**考虑特征之间的关系**。

1. 新任务或新数据集：P-Tune可以将预训练模型微调到新的任务或数据集上，从而提高模型的准确性和泛化能力。
2. 领域自适应：在模型需要适应新的领域时，P-Tune可以调整模型参数，从而使模型更好地适应新的数据分布。
3. 模型压缩：通过微调模型参数，P-Tune可以压缩模型大小，从而减少模型的计算复杂度和存储空间，提高模型的运行效率。





## Reparameterization-Based Methods 重新参数化方法

Reparameterization-Based方法专注于找到与基础模型中的相同权重矩阵的低维表示。在Hu等人的工作中首次揭示了微调和低维表示之间的联系。作者们将模型的完整参数与较低维度的表示进行了联系。根据任务的不同，作者们能够使用大约0.0002%的可训练参数实现完全微调模型结果的90%。

### LORA

https://huggingface.co/docs/peft/conceptual_guides/lora

在微调中，一种基于重新参数化的流行技术是称为低秩适应（LoRa）的方法，LoRa通过学习一个独立的矩阵来更新权重矩阵，该矩阵表示来自优化的更新。他们更进一步创建了两个较小维度的权重矩阵来表示这种差异。通过创建较小维度的权重矩阵，我们需要学习的参数更少。



在LoRa中，

- 冻结大部分原来LLM的权重
- 注入两个秩分解矩阵，并训练

![image-20230914210329870](https://cdn.jsdelivr.net/gh/631068264/img/202309142103917.png)



我们选择将所有的更新隔离到一个单独的矩阵中。这个矩阵，我们将其表示为ΔW，表示在微调过程中我们学习到的所有参数更新。

![img](https://cdn.jsdelivr.net/gh/631068264/img/202309091304995.png)

让我们将**W_0分配为具有dxk维度（d行k列）的矩阵**。我们希望更新它的参数，使其与我们的新目标对齐。您可以用ΔW来表示对参数的更新，它也具有dxk的维度。我们可以使用下面的方程来建模我们的更新规则。

$$W_0\;\;\;+\Delta W=W_0\;\;\;+AB $$

**为什么AB比DeltaW更好的原因**：矩阵A只有dxr的维度，而矩阵B有rxk的维度。**如果我们选择一个非常小的数字r（典型值为8  4,8。。64）**，那么矩阵A和B中的参数数量要远远小于ΔW。如果我们只学习矩阵A和B的参数，那么我们**将少学习d*k-d*r-rk个参数**。实际上，这使我们只需学习原网络参数的0.1-0.5%。

![image-20230914210710178](https://cdn.jsdelivr.net/gh/631068264/img/202309142107284.png)

通常情况下，我们将这个**更新规则应用于Transformer block 里面self attention的K，V矩阵**，我们还添加了一个缩放因子，设置为1/r，以调整更新的信息量。

```python
def lora_linear(x, W):
    scale = 1 / r  # r is rank
    h = x @ W
    h += x @ W_a @ W_b  # W_a,W_b determined based on W  加上原来的权重
    return scale * h

def self_attention_lora(x):
    """ Pseudo code from Lialin et al. [2]."""

    k = lora_linear(x, W_k)
    q = x @ W_q
    v = lora_linear(x, W_v)
    return softmax(q @ k.T) @ v
```

参数调优：**模型垂直性越大，r越大**

![image-20230914211355202](https://cdn.jsdelivr.net/gh/631068264/img/202309142113316.png)

和**full fine-tune微调相比：**：训练参数减少，使用更少显存，精度略低，**相同推理延迟**，重新训练某些层，冻结主要权重。



![image-20230914211325045](https://cdn.jsdelivr.net/gh/631068264/img/202309142113097.png)

这里的要点是,**r选择4-32**的秩范围可以在减少可训练参数和保持性能之间提供一个良好的折衷。



**Qlora是对量化模型进行lora**

### 对比LORA和P-Tune

对于领域化的数据定制处理，P-Tune（Parameter Tuning）更加合适。

领域化的数据通常包含一些特定的领域术语、词汇、句法结构等，与通用领域数据不同。对于这种情况，微调模型的参数能够更好地适应新的数据分布，从而提高模型的性能。

相比之下，LORA（Layer-wise Relevance Propagation）更注重对模型内部的特征权重进行解释和理解，通过分析模型对输入特征的响应来解释模型预测的结果。虽然LORA也可以应用于领域化的数据定制处理，但它更适合于解释模型和特征选择等任务，而不是针对特定领域的模型微调。

因此，对于领域化的数据定制处理，建议使用P-Tune方法。

两者对于低资源微调大模型的共同点都是冻结大模型参数，通过小模块来学习微调产生的低秩改变。但目前存在的一些问题就是这两种训练方式很容易参数灾难性遗忘，因为模型在微调的时候整个模型层参数未改变，而少参数的学习模块微调时却是改变量巨大，容易给模型在推理时产生较大偏置，使得以前的回答能力被可学习模块带偏，在微调的时候也必须注意可**学习模块不能过于拟合微调数据**，否则会丧失原本的预训练知识能力，产生灾难性遗忘。最好能够在**微调语料中也加入通用学习语料一起微调，避免产生对微调语料极大的偏向**

### lora 参数

[Finetuning LLMs with LoRA and QLoRA: Insights from Hundreds of Experiments](https://lightning.ai/pages/community/lora-insights/)

![image-20231102221136300](https://cdn.jsdelivr.net/gh/631068264/img/202311022213436.png)

- LoRA 最重要的参数之一是“r”，它决定了 LoRA 矩阵的秩或维数，直接影响模型的复杂性和容量。较高的“r”意味着更强的表达能力，但可能导致过度拟合，而较低的“r”可以减少过度拟合，但会牺牲表达能力。

- 较高的“alpha”会更加强调低秩结构或正则化，而较低的“alpha”会减少其影响，使模型更加依赖于原始参数。调整“alpha”有助于在拟合数据和通过正则化模型防止过度拟合之间取得平衡。

- 迭代次数的增加可能会导致整体性能变差。假设是数据集不包含任何相关的算术任务，并且当模型更多地关注其他任务时，它会主动忘记基本算术，导致算术基准的下降最为显着。

  





## 方法对比

![img](https://cdn.jsdelivr.net/gh/631068264/img/202309091341104.png)



# RLHF(Reinforcement Learning from Human Feedback)

人类反馈的强化学习

通过使模型与人类反馈保持一致，并使用强化学习作为算法，减少有害信息。这些重要的人类价值观，有用（Helpful）、诚实(Honest)和无害(HarmLess)，有时被统称为HHH

![image-20230916155151960](https://cdn.jsdelivr.net/gh/631068264/img/202309161551023.png)

强化学习是一种机器学习类型，智能体人在环境中采取行动，以达到最大化某种累积奖励的目标，从而学习做出与特定目标相关的决策。通过迭代这个过程，智能体逐渐改进其策略或政策，做出更好的决策，增加其成功的机会。

![image-20230916155652657](https://cdn.jsdelivr.net/gh/631068264/img/202309161556703.png)

在RLHF中，指导动作的Agent策略是LLM，其

- **目标是生成被认为与人类偏好相符的文本，** The goal is to generate text that is considered aligned with human preferences.

- **环境是模型的上下文窗口，可以通过提示在其中输入文本的空间**   The environment is the context window of the model, where text can be inputted through prompts.

- 模型在采取行动之前考虑的**状态是当前的上下文**。 The state that the model considers before taking action is the current context.

- **动作是生成文本的行为**，动作空间是Token词汇，意味着模型可以选择生成输出结果的所有可能Token。在任何特定时刻，模型将采取的动作，也就是它接下来将选择哪个Token,取决于上下文中的提示文本和词汇空间的概率分布。 The action is the behavior of generating text

- **奖励是根据输出内容的与人类偏好的对齐程度来分配的** Rewards are allocated based on the alignment of the output content with human preferences.

- 为了减少人工成本，可以使用一个额为model称为**奖励模型，来分类LLM的输出并评估与人类偏好的对齐程度。**可以从少量的人类示例开始，用你的传统监督学习方法训练次级模型。一旦训练完成，你可以**使用奖励模型来评估LLM的输出并分配一个奖励值。这个奖励值反过来又被用来更新LLM的权重并训练一个新的符合人类偏好的版本。** 它编码了所有从人类反馈中学到的偏好，在模型在多次迭代中更新其权重的方式中起着核心作用

  reward model can be used to classify the outputs of LLM,evaluate their alignment with human preferences. Once trained, the reward model can be used to evaluate the outputs of the language model and assign a reward value. This reward value is then used to update the weights of the LLM and train a new version aligned with human preferences

- 行动和状态的序列被称为展开(Rollout)

  ![image-20230916161533240](https://cdn.jsdelivr.net/gh/631068264/img/202309161615326.png)



## RLHF数据准备

“提示词-生成结果”数据集由多条提示词组成，每个提示词都由LLM处理以产生一组生成结果。

![image-20230916162011903](https://cdn.jsdelivr.net/gh/631068264/img/202309161620949.png)

**收集人类标注员对 LLM 生成的生成结果的反馈**

- 你必须决定你希望人类根据什么标准来评告 LLM 的生成结果    defined  the criterion
- 一旦你定义好了标准，就可以要求标注员根据这个标准评估数据集中的每条生成结果，进行排名。Once you have defined the criteria, you can rank each generated result in the dataset according to those criteria.
- 同样的“提示词-生成结果”数据集通常会分配给多个人类标注员，以确保大家的答案更一致，这样可以降低某个人标记不准确的风险。The same "prompt-response" dataset is typically assigned to multiple human annotators to ensure greater consistency in their answers and reduce the risk of inaccurate labeling by any individual.

![image-20230916163655188](https://cdn.jsdelivr.net/gh/631068264/img/202309161636274.png)



一旦人类标注员完成了对“提示词-生成结果”数据集的评估，你就可以用这些数据来训练奖励模型了，训练开始前，你需要将排名数据转化为生成结果之间的两两对比。然后你可以重新对提示词排序，使排名高的项排在前面。

you can rerank the prompts based on their rankings

![image-20230916164117404](https://cdn.jsdelivr.net/gh/631068264/img/202309161641458.png)

数据整理完毕后，你将得到一个适合训练奖励模型的人类反馈格式

虽然点赞和点踩的反馈通常比排名反馈更容易收集，但排名反馈可以给你提供更多的“提示词生成结果”数据来训练你的奖励模型。



## 奖励模型

奖励模型将有效地取代人类的数据标注员，并自动选择在RLHF过程中自动挑选最佳的结果。

![image-20230916165252789](https://cdn.jsdelivr.net/gh/631068264/img/202309161652874.png)

将奖励模型用作二进制分类器，为每个提示完成对提供奖励值

![image-20230916165503196](https://cdn.jsdelivr.net/gh/631068264/img/202309161655248.png)

![image-20230916165527634](https://cdn.jsdelivr.net/gh/631068264/img/202309161655685.png)



## 使用奖励模型微调LLM

- 从你的提示词数据集中选择一个提示词，将提示词输入LLM，然后生成一个补全 (Completion)
- 接下来，把这个补全和原始的提示词一起，作为一个“提示词-补全”对发送到奖励模型
- 奖励模型根据训练好的人类反馈模型估这个“提示词-补全”对，并返回一个奖励值，
- 然后把这个奖励值传递给“提示词-补全”对强化学习算法，更新LLM的权重，并使其朝着生成更一致、更高奖励的生成结果的方向移动。让我们把这个模型的中间版本称为**RL更新的LLM**。
- 随着迭代模型越来越好，你会看到每次迭代后奖励分数都在提高，你将继续这个送代过程，直到你的模型的对齐结果达到一定的评估标准。你也可以定义一个最大的步数。

![image-20230916173317896](https://cdn.jsdelivr.net/gh/631068264/img/202309161733945.png)



强化学习算法的确切性质。

这是一个算法，它接收奖励模型的输出，并使用它来更新LLM模型的权重，使得奖励分数随着时间的推移而增加。

而近端策略优化(也称为PPO  Proximal Policy Optimization) 是其中的热门算法。

## PPO

[大型语言模型与生成式AI——人类反馈强化学习7——PPO增强学习算法深度解析](https://www.youtube.com/watch?v=pabhSqmEaaI&list=PLiuLMb-dLdWL4KBaU3FTM5f_oMcSvXcZw&index=29&ab_channel=%E5%AE%9D%E7%8E%89%E7%9A%84%E6%8A%80%E6%9C%AF%E5%88%86%E4%BA%AB)

### 第一阶段确定的损失和奖励

**价值函数估计**给定状态S的**预期总奖励**，换句话说，当LLM生成完成的每一个词元时，你想要估计基于当前词元序列的未来总奖励。这可以被视为一个基线，用来根据你的对齐标准评估完成的质量。



目标是最小化价值损失，即实际未来总奖励 (在这个例子中是1.87)和它在价值函数中的近似值(在这个例子中是1.23) 之间的差异。价值损失使得对未来奖励的估计更准确。

![image-20230917101940118](https://cdn.jsdelivr.net/gh/631068264/img/202309171019236.png)



### 第二阶段用于更新权重

对模型进行小幅更新并评估这些更新对你的模型对齐目标的影响。

模型权重的更新是由提示词-生成结果、损失和奖励来指导的。PPO还确保将模型更新保持在一个叫做信任区域的小区域内，这就是PPO的近似方面发挥作用的地方。理想情况下，这一系列小的更新会将模型向更高的奖励移动。



**目标是找到一个预期奖励高的策略，你试图对LLM权重进行更新。使得完成的结果更符合人类的偏好，因此获得更高的奖励。策略损失是PPO算法在训练过程中试图优化的主要目标。**



在这个LLM的上下文中，A_t在S_t下的Pi，是下一个词元A_t在当前提示词S_t下的概率。动作A_t是下一个词元，状态S_t是到词元t的完成提示词。

分母是下一个词元的概率（初始版本LLM已经冻结权重），分子是下一个词元的概率（通过更新的LLM，我们可以改变以获得更好的奖励）A-hat_t被称为给定动作选择的估计优势项，优势项**估计当前动作与数据状态下所有可能动作相比是好还是差**

![image-20230917114932986](https://cdn.jsdelivr.net/gh/631068264/img/202309171149036.png)

相当于：**你有一个提示词S，你有不同的路径来完成它，通过图中的不同路径来说明。优势项告诉你当前的词元A t相对于所有可能的词元有多好或多坏**，顶部路径是更好的生成结果，得到更高的奖励，底部路径是最差的生成结果。positive优势意味着建议的词元优于平均水平，增加当前词元的概率似乎是一种能带来更高回报的好策略。

![image-20230917115730002](https://cdn.jsdelivr.net/gh/631068264/img/202309171157051.png)

![image-20230917121005696](https://cdn.jsdelivr.net/gh/631068264/img/202309171210823.png)

**直接最大化表达式会导致问题**，因为我们的计算，是在我们的优势估计有效的假设下可靠的。**只有当旧的和新的政策接近时，优势估计才有效。**



注意，这第二个表达式定义了一个区域，两个政策接近彼此的地方。这些额外的条款是防护栏，简单地定义了一个接近LLM的区域，**这被称为信任区域**。

![image-20230917122557577](https://cdn.jsdelivr.net/gh/631068264/img/202309171225630.png)

总的来说，优化PPO政策目标会得到一个更好的LLM，而不会超出不可靠的区域。



## Calculate entropy loss

虽然政策损失将模型向对齐目标移动，熵允许模型保持创造性

![image-20230917123636906](https://cdn.jsdelivr.net/gh/631068264/img/202309171236958.png)

通过一定加权相加

![image-20230917123830777](https://cdn.jsdelivr.net/gh/631068264/img/202309171238828.png)



## 奖励攻击

代理通过选择那些使其获得最大奖励的行为来欺骗系统，即使这些行动并不符合原始的目标。**LLM生成满足奖励模型，但是毫无意义，毫无用处的补全。**

你可以使用最初的LLM作为性能参考。在训练过程中，每个提示词都会传递给两个模型，由参考模型和RL更新模型同时生成内容，然后，你可以比较两个生成结果（KL 散度是一个统计度量，用来衡量两个概率分布有多不同）

In the training process, each prompt is passed to two models, the reference model and the RL update model, for content generation.You can compare the generated results from both models ,penalized

相当于添加了一个惩罚项到奖励计算中，如果RL更新模型与参考LLM差距太大，并产生两个截然不同的完成结果，将对其进行处罚

![image-20230916193350378](https://cdn.jsdelivr.net/gh/631068264/img/202309161933429.png)

你只更新路径适配器的权重，而不是 LLM 的全部权重。这意味着你可以重用同一个底层的LLM 作为参考模型和 PPO 模型，你用训练路径参数进行更新。这将训练过程中的内存占用减少大约一半

![image-20230916193655254](https://cdn.jsdelivr.net/gh/631068264/img/202309161936307.png)



## 有害评估

首先，你将为原始的LLM创建一个有害信息的基准分数，通过使用一个可以评估有害信息的奖励模型来评价其在摘要数据集上的完成情况。然后，你将在同一数据集上评估新的人类对齐模型，并比较分数

## Constitutional Al 

宪法 AI 是一种根据一套管理模型行为的规则和原则来训练模型的策略。与样本提示词结合，这些共同构建了模型的宪法。接着，你会教模型自我评估并根据这些准则调整生成结果。宪法 AI 不仅对扩大反馈有用，它还可以帮助解决 RLHF 的一些意外后果。

向模型 提供一组宪法准则有助于模型在这些冲突的利益之间找到平衡，并降低伤害。

你可以告诉模型选择最有帮助、最诚实和最无害的回应，要求模型通过评估其生成结果是否鼓励非法、不道德或其他不当的活动来优先考虑无害性。

在实施宪法 AI 方法时，你需要在两个不同的阶段训练你的模型。**训练方法**

- 在第一阶段，你进行监督学习，尝试让模型产生有害的回应，这个过程被称为红队操作，随后指示模型根据宪法原则对其有害回应进行自我评估，并修改它们以符合这些规则。完成这步后。可以利用红队的提示词 和 经过修订过的遵循宪法的生成结果 来对模型进行微调。

  ![image-20230916201727638](https://cdn.jsdelivr.net/gh/631068264/img/202309162017736.png)

  

  ![image-20230916202117756](https://cdn.jsdelivr.net/gh/631068264/img/202309162021895.png)

  最后使用这对提示词-符合宪法生成的结果作为训练数据

  ![image-20230916202945021](https://cdn.jsdelivr.net/gh/631068264/img/202309162029081.png)

- 这个阶段类似于RLHF，只是我们现在使用的反馈是由模型生成的，而不是人类的反馈

  你可以利用先前微调的模型对你的提示词生成一组响应，并询问模型哪个响应更加遵循宪法原则。

  这会产生一个模型偏好的数据集，你可以利用它来训练奖励模型。

  然后可以使用像PPO这样的强化学习算法进一步微调你的模型。

  ![image-20230916203940365](https://cdn.jsdelivr.net/gh/631068264/img/202309162039417.png)







# 模型压缩

对模型进行压缩的目标
- 减少模型大小
- 加快**推理速度**
- 保持相同精度



## 量化

模型量化是一种将浮点计算转成低比特定点计算的技术，可以有效的降低模型计算强度、参数大小和内存消耗，但往往带来巨大的精度损失。尤其是在极低比特(<4bit)、二值网络(Ibit)、甚至将梯度进行量化时，带来的精度挑战更大。

![image-20230910164304742](https://cdn.jsdelivr.net/gh/631068264/img/202309101643784.png)

**好处**

- 减少内存，显存，存储占用：FP16的位宽是FP32的一半，因此权重等参数所占用的内存也是原来的一半，节省下来的内存可以放更大的网络模型或者使用更多的数据进行训练。
- 加快通讯效率：针对分布式训练，特别是在大模型训练的过程中，通讯的开销制约了网络模型训练的整体性能，通讯的位宽少了意味着可以提升通讯性能，减少等待时间，加快数据的流通。
- 计算效率更高：低比特的位数减少少计算性能也更高，INT8 相对比 FP32的加速比可达到3倍甚至更高
- 降低功耗
- **支持微处理器**，有些微处理器属于8位的，低功耗运行浮点运算速度慢，需要进行8bit量化，**让模型在边缘集群或终端机器运行**。

**坏处**

- 精度损失，推理精度确实下降
- 硬件支持程度，不同硬件支持的低比特指令不相同。不同硬件提供不同的低比特指令计算方式不同( PFI6、HF32)。不同硬件体系结构Kernel优化方式不同
- 软件算法是否能加速，混合比特量化需要进行量化和反向量，插入 Cast 算子影响 kernel 执行性能。降低运行时内存占用，与降低模型参数量的差异。模型参数量小，压缩比高，不代表执行内存占用少。

### 量化原理

浮点数用整数，建立映射关系。

![image-20230910170757986](https://cdn.jsdelivr.net/gh/631068264/img/202309101707028.png)

**图例**映射到int8，上面是浮点数最大最小值，浮点数范围可以截断，超过就不映射。



量化类型分**对称和非对称**，是否根据0轴是对称轴， int uint

![image-20230910171424297](https://cdn.jsdelivr.net/gh/631068264/img/202309101714394.png)

**量化公式** int8作为实例

![image-20230910171900745](https://cdn.jsdelivr.net/gh/631068264/img/202309101719841.png)

![image-20230910171937019](https://cdn.jsdelivr.net/gh/631068264/img/202309101719115.png)

### 量化方法
- 训练后量化 (post-training quantization，PTQ)
- 训练中量化 (quantization-aware training，QAT)
  - 训练时候做前向和反向传播 ： 前向使用量化后的低精度计算，反向使用原来浮点数权重和梯度
- 量化训练(Quant Aware Training, QAT)量化训练让模型感知量化运算对模型精度带来的影响，通过 finetune 训练降低量化误差
- 静态离线量化(Post Training Quantization Static, PTQ Static)静态离线量化使用少量无标签校准数据，采用 KL 散度等方法计算量化比例因子
- 动态离线量化(Post Training Quantization Dynamic, PTQ Dynamic)动态离线量化仅将模型中特定算子的权重从FP32类型映射成 INT8/16 类型 （直接走量化，不管误差）

![image-20230910170405460](https://cdn.jsdelivr.net/gh/631068264/img/202309101704499.png)

### QAT

https://github.com/chenzomi12/DeepLearningSystem/blob/main/043INF_Slim/03.qat.pdf

感知量化训练( Aware Quantization Training)，**通常在密集计算，激活函数，输入输出的地方插入fake quant**

模型中插入**伪量化节点fake quant**来**模拟量化引入的误差**。端测推理的时候折叠fake quant节点中的属性到tensor中，在端测推理的过程中直接使用tensor中带有的量化属性参数。

![image-20230910172929711](https://cdn.jsdelivr.net/gh/631068264/img/202309101729812.png)

算法流程

- 预训练模型改造网络模型，插入fake quant
- 用大量标签数据和改造后的模型，进行train/fine tuning
- 通过学习把量化误差降低
- 最后输出QAT model
- QAT model通过转换模块，去掉冗余fake quant，得到量化后模型





**伪量化节点的作用**

- 找到**输入数据的分布**，即找到 min 和 max 值 ;
- 模拟量化到低比特操作的时候的精度损失，把该损失作用到网络模型中，传递给损失函数。**让优化器去在训练过程中对该损失值进行优化。**

![image-20230910174033974](https://cdn.jsdelivr.net/gh/631068264/img/202309101740053.png)

![image-20230910192127405](https://cdn.jsdelivr.net/gh/631068264/img/202309101921453.png)

![image-20230910192502852](https://cdn.jsdelivr.net/gh/631068264/img/202309101925899.png)

![image-20230910192957483](https://cdn.jsdelivr.net/gh/631068264/img/202309101929597.png)



### PTQ

#### PTQ Dynamic

动态离线量化 ( Post Training Quantization Dynamic, PTQ Dynamic)

- 仅将模型中**特定算子的权重**从FP32类型映射成 INT8/16 类型
- 主要可以减小模型大小，对特定加载权重费时的模型可以起到一定加速效果
- 但是对于**不同输入值，其缩放因子是动态计算**，因此动态量化是几种量化方法中性能最差的



权重量化成 INTI6 类型，模型精度不受影响，模型大小为原始的 1/2。
权重量化成 INT8 类型，模型精度会受到影响，模型大小为原始的 1/4.



算法流程

- 对预训练模型的FP32权重转换成int8，就可以得到量化模型

#### PTQ Static

静态离线量化( Post Training Quantization Static, PTQ Static)

- 同时也称为校正量化或者数据集量化。使用少量无标签校准数据，核心是计算量化比例因子使用静态量化后的模型进行预测，在此过程中量化模型的缩放因子会根据输入数据的分布进行调整。

- 静态离线量化的目标是求取量化比例因子，主要通过对称量化、非对称量化方式来求，而找最大值或者闯值的方法又有MinMax、KLD、ADMM、EQ等方法

算法流程

- 预训练模型改造网络模型，插入fake quant

- 用少量非标签数据和改造后的模型，进行校正

- 通过量化

- 最后输出PTQ model

![image-20230910202422921](https://cdn.jsdelivr.net/gh/631068264/img/202309102024985.png)

![image-20230910202436704](https://cdn.jsdelivr.net/gh/631068264/img/202309102024763.png)

![image-20230910203059737](https://cdn.jsdelivr.net/gh/631068264/img/202309102030808.png)

### 量化部署

![image-20230910203601561](https://cdn.jsdelivr.net/gh/631068264/img/202309102036600.png)

![image-20230910203650733](https://cdn.jsdelivr.net/gh/631068264/img/202309102036803.png)

量化

![image-20230910203737479](https://cdn.jsdelivr.net/gh/631068264/img/202309102037521.png)

反量化

![image-20230910204437120](https://cdn.jsdelivr.net/gh/631068264/img/202309102044161.png)

重量化

![image-20230910204314687](https://cdn.jsdelivr.net/gh/631068264/img/202309102043727.png)









## 剪枝

**量化剪枝的区别**

- 模型量化是指通过减少权重表示或激活所需的比特数来压缩模型
- 模型剪枝研究模型权重中的几余，并尝试删除/修剪几余和非关键的权重

![image-20230910205322275](https://cdn.jsdelivr.net/gh/631068264/img/202309102053316.png)

剪枝的功效

- 在内存占用相同情况下，大稀疏模型比小密集模型实现了更高的精度 （**所以一般不训练小模型，而是训练大模型再剪枝**）
- 经过剪枝之后稀疏模型要优于，同体积非稀疏模型
- 资源有限的情况下，剪枝是比较有效的模型压缩策略
- 优化点还可以往硬件稀疏矩阵储存方向发展。

### 剪枝分类

![image-20230910210413270](https://cdn.jsdelivr.net/gh/631068264/img/202309102104364.png)

Unstructured Pruning(非结构化剪枝）

- 优点: 剪枝算法简单，模型压缩比高
- 缺点: 精度不可控，剪枝后权重矩阵稀疏，没有专用硬件难以实现压缩和加速的效果

Structured Pruning ( 结构化剪枝）

- 优点:大部分算法在 channel 或者 ayer 上进行剪枝，保留原始卷积结构，不需要专用硬件来实现
- 缺点: 剪枝算法相对复杂

**压缩率 VS 性能**

**Fine-grained Pruning (细粒度剪枝)** (需要高压缩率和较小性能损失的场景，如移动设备或边缘计算。)

- **灵活的剪枝指标**：能够根据权重的重要性灵活选择剪枝的对象。
- **更高的压缩比**：由于可以精确识别冗余权重，通常能实现更大的模型压缩。

**Pattern-based Pruning (模式化剪枝)** (适用于需要在不显著牺牲性能的前提下增加模型计算效率的场合)

- **N:M 稀疏性**：在每个连续的 M 个元素中，有 N 个元素被剪枝
- **硬件支持**：NVIDIA 的 Ampere GPU 架构支持这种稀疏性，能够实现约 2 倍的加速效果。
- **保持模型准确性**：经过多种任务测试，通常能够维持较高的准确率。



在从神经网络模型中移除参数时，以下原则是关键：

- **去除不重要的参数**：越不重要的参数被移除，剪枝后的神经网络性能通常越好。

- 定义重要性

  - 基于权重的绝对值，认为绝对值较大的权重比其他权重更重要 

  - 不同维度，和衡量方式

    ![image-20241009161448111](https://cdn.jsdelivr.net/gh/631068264/img/202410091627151.png)

    ![image-20241009161511454](https://cdn.jsdelivr.net/gh/631068264/img/202410091615845.png)

    ![image-20241009161611709](https://cdn.jsdelivr.net/gh/631068264/img/202410091616788.png)





### 剪枝流程

- 训练一个模型 -> 对模型进行剪枝 -> 对剪枝后模型进行微调
- 在模型训练过程中进行剪枝 ->对剪枝后模型进行微调
- 进行剪枝 ->从头训练剪枝后模型



训练 Training : 训练过参数化模型，得到最佳网络性能，以此为基准

剪枝 Pruning:根据算法对模型剪枝，调整网络结构中通道或层数，得到剪枝后的网络结构

微调 Finetune:在原数据集上进行微调，用于重新弥补因为剪枝后的稀疏模型丢失的精度性能



### 剪枝算法

使用 LI-norm 标准来衡量卷积核的重要性，LI-norm 是一个**很好的选择卷积核**的方法，认为如果一个flter的绝对值和比较小，说明该filter并不重要。论文指出对剪枝后的网络结构从头训练要比对重新训练剪枝后的网络



算法步骤

![](https://cdn.jsdelivr.net/gh/631068264/img/202309102140443.png)

### Finding Pruning Ratios (寻找剪枝比例)

**敏感性分析的重要性**

分层敏感性：不同层对剪枝的敏感性不同，因此需要为每一层设定不同的剪枝比例。

- **更敏感的层**：例如，网络的第一层通常对剪枝较为敏感，去除其参数可能导致显著的性能下降。
- **冗余层**：某些层可能包含冗余参数，可以更激进剪枝而不会影响整体性能





**敏感性分析流程**

**选择层进行分析**：逐层分析模型的敏感性。

**评估剪枝影响**：对每一层进行不同的剪枝比例测试，观察其对模型准确率的影响。

**确定剪枝比例**

- 记录每层在不同剪枝比例下的性能变化。
- 根据准确率的变化选择合适的剪枝比例，以实现最佳的模型压缩效果。



**传统方法通常未考虑不同层之间的相互影响，这可能导致剪枝决策不够理想。**









## 知识蒸馏 Knowledge Distillation

旨在把一个大模型或者多个模型学到的知识迁移到另一个轻量级单模型上，方便部署。简单的说就是用新的小模型去学习大模型的预测结果，改变一下目标函数。

Student Model 学生模型模仿 Teacher Model 教师模型，二者相互竞争，直到学生模型可以与教师模型持亚甚至卓越的表现:

知识蒸馏的算法，主要由:

- 知识 Knowledge
- 蒸算法 Distillate
-  师生架构

![image-20230910215834824](https://cdn.jsdelivr.net/gh/631068264/img/202309102158898.png)

### 知识的来源

![image-20230910220624308](https://cdn.jsdelivr.net/gh/631068264/img/202309102206354.png)

#### Response-Based Knowledge

主要指Teacher Model 教师模型输出层的特征主要思想是让 Student Model **学生模型直接学习教师模式的预测结果**( Knowledge )

Response-based knowledge 主要指 Teacher Model 教师模型最后一层--**输出层的特征**。其主要思想是让学生模型直接模仿教师模式的最终预测:

![image-20230910221848826](https://cdn.jsdelivr.net/gh/631068264/img/202309102218901.png)

-  Teacher Model 教师模型最后一层特征
- 通过Distillation Loss，让学生的最后一层特征学习教师模型最后一层特征
- 通过loss，使得两个logits越小越好

#### Feature-Based Knowledge

深度神经网络善于学习到不同层级的表征，因此中间层和输出层的都可以被用作知识来训练学生模型。

中间层学习知识的 Feature-Based Knowledge 对于 Response-Based Knowledge是一个很好的补充，其主要思想是将教师和学生的特征激活进行关联起来。

虽然基于特征的知识迁移为学生模型的学习提供了良好的信息，但如何有效地从教师模型中选择提示层，从学生模型中选择引导层，仍有待进一步研究。由于提示层和引导层的大小存在显著差异，如何正确匹配教师和学生的特征表示也需要探讨

![image-20230910222601213](https://cdn.jsdelivr.net/gh/631068264/img/202309102226394.png)

#### Relation-Based Knowledge

基于 Feature-Based Knowledge 和 Response-Based Knowledge 知识都**使用了教师模型中特定层中特征的输出**。基于关系的知识进一步探索了不同层或数据样本之间的关系。

![image-20230910222905540](https://cdn.jsdelivr.net/gh/631068264/img/202309102229616.png)

loss 不仅仅学习刚才讲到的网络模型中间的特征，最后一层的特征，还学习数据样本和网络模型层之间的关系

### 蒸馏方法

![image-20230910223949315](https://cdn.jsdelivr.net/gh/631068264/img/202309102239392.png)

#### Offline Distillation

大多数蒸馏采用 Ofine Distillation，蒸留过程被分为两个阶段:

- 蒸馏前教师模型预训练;
- 蒸馏蒸留算法迁移知识。

因此 Ofline Distillation 主要侧重于知识迁移部分通常采用单向知识转移和两阶段训练过程。在步骤1) 中**需要教师模型参数量比较大**，训练时间比较长，这种方式对学生模型的蒸馏比较高效。**缺点：学生过于依赖老师模型**

#### Online Distillation

Online Distilation 主要针对参数量大、精度性能好的**教师模型不可获得**的情况。

**教师模型和学生模型同时更新**，整个知识蒸馏算法是一种有效的端到端可训练方案。

**缺点**: 现有的 Online Distilation 往往难以获得在线环境下参数量大、精度性能好的教师模型



**Self Distillation** 是online的一个特例，**教师模型和学生模型使用相同的网络结构，同样采样端到端可训练方案**

### 蒸馏算法Hinton

![image-20230910225209002](https://cdn.jsdelivr.net/gh/631068264/img/202309102252072.png)

T越大，负标签信息会被放大

![image-20230910225516394](https://cdn.jsdelivr.net/gh/631068264/img/202309102255464.png)

![image-20230910225645351](https://cdn.jsdelivr.net/gh/631068264/img/202309102256423.png)

![image-20230910225942095](https://cdn.jsdelivr.net/gh/631068264/img/202309102259170.png)

![image-20230910230654808](https://cdn.jsdelivr.net/gh/631068264/img/202309102306884.png)

![image-20230910230738864](https://cdn.jsdelivr.net/gh/631068264/img/202309102307935.png)

![image-20230910230752378](https://cdn.jsdelivr.net/gh/631068264/img/202309102307460.png)

![image-20230910230830183](https://cdn.jsdelivr.net/gh/631068264/img/202309102308258.png)
