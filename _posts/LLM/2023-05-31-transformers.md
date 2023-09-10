---
layout:     post
rewards: false
title:   transformers
categories:
    - AI
tags:
   - 大模型

---



# 概念

## 预训练

是从头开始训练模型的行为：随机初始化权重，并且在没有任何先验知识的情况下开始训练。这种预训练通常是在非常大量的数据上进行的。因此，它需要非常大的语料库，训练可能需要长达数周的时间。

![语言模型的预训练在时间和金钱上都是昂贵的。](https://huggingface.co/datasets/huggingface-course/documentation-images/resolve/main/en/chapter1/pretraining.svg)

## 微调

是在对模型进行预训练**之后**进行的训练。要执行微调，您首先需要一个预训练的语言模型，然后使用特定于您的任务的数据集执行额外的训练。基于预训练模型微调，只需要有限数量的数据：预训练模型获得的知识是“迁移”的，因此称为*迁移学习*。

![语言模型的微调在时间和金钱上都比预训练便宜。](https://cdn.jsdelivr.net/gh/631068264/img/202305312242820.svg)

## Transformers

该模型主要由两个块组成：

- **编码器（左）**：编码器接收输入并构建它的表示（其特征）。这意味着模型经过优化以从输入中获取理解。

- **解码器（右）**：解码器使用编码器的表示（特征）和其他输入来生成目标序列。这意味着该模型针对生成输出进行了优化。

  ![](https://cdn.jsdelivr.net/gh/631068264/img/202306010950783.svg)

  这些部分中的每一个都可以独立使用，具体取决于任务：

  - **仅编码器模型**：适用于需要理解输入的任务，例如句子分类和命名实体识别。
  - **仅解码器模型**：适用于文本生成等生成任务。
  - **编码器-解码器模型**或**序列到序列模型**：适用于需要输入的生成任务，例如翻译或摘要。



## 注意层

只需要知道这一层会告诉模型在处理每个单词的表示时特别注意你传递给它的句子中的某些单词（并或多或少地忽略其他单词）。

在每个阶段，编码器的注意力层可以访问初始句子中的所有单词，而解码器的注意力层只能访问输入中给定单词之前的单词。

# pipeline

 Transformers 库中最基本的对象是函数`pipeline()`。它将模型与其必要的预处理和后处理步骤连接起来，使我们能够直接输入任何文本并获得可理解的答案：

```py
from transformers import pipeline

classifier = pipeline("sentiment-analysis")
classifier("I've been waiting for a HuggingFace course my whole life.")
```



```
[{'label': 'POSITIVE', 'score': 0.9598047137260437}]
```

多个input

```py
classifier(
    ["I've been waiting for a HuggingFace course my whole life.", "I hate this so much!"]
)
```



```
[{'label': 'POSITIVE', 'score': 0.9598047137260437},
 {'label': 'NEGATIVE', 'score': 0.9994558095932007}]
```

默认情况下，此**pipeline**选择一个特定的预训练模型，该模型已针对英语情感分析进行了微调。创建`classifier`对象时会下载并缓存模型。如果重新运行该命令，将使用缓存的模型，无需再次下载模型。

将一些文本传递到管道时涉及三个主要步骤：

- 文本被预处理为模型可以理解的格式。**使用分词器进行预处理**

- 预处理的输入被传递给模型。

- 模型的预测经过后处理，因此您可以理解它们。

管道将三个步骤组合在一起：预处理、通过模型传递输入和后处理：

![完整的 NLP 管道：文本标记化、ID 转换以及通过 Transformer 模型和模型头进行推理。](https://cdn.jsdelivr.net/gh/631068264/img/202306011053329.svg)

```py
raw_inputs = [
    "I've been waiting for a HuggingFace course my whole life.",
    "I hate this so much!",
]
inputs = tokenizer(raw_inputs, padding=True, truncation=True, return_tensors="pt")
print(inputs)

# input_ids包含两行整数（每个句子一个），它们是每个句子中标记的唯一标识符
# attention_mask 输入 ID 张量完全相同形状的张量，填充有 0 和 1：1 表示应注意相应的标记，0 表示不应注意相应的标记（即它们应该被忽略）模型的注意力层
{
    'input_ids': tensor([
        [  101,  1045,  1005,  2310,  2042,  3403,  2005,  1037, 17662, 12172, 2607,  2026,  2878,  2166,  1012,   102],
        [  101,  1045,  5223,  2023,  2061,  2172,   999,   102,     0,     0,     0,     0,     0,     0,     0,     0]
    ]), 
    'attention_mask': tensor([
        [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
        [1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0]
    ])
}
```





Transformer 模块输出的向量通常很大。它一般具有三个维度：

- **批量大小**：一次处理的序列数（在我们的示例中为 2）。
- **序列长度**：序列的数字表示的长度（在我们的示例中为 16）。
- **Hidden size** : 每个模型输入的向量维度。

由于最后一个值，它被称为“高维”。隐藏大小可能非常大（对于较小的模型，768 是常见的，而在较大的模型中，这可以达到 3072 或更多）。

如果我们将预处理的输入提供给我们的模型，我们可以看到这一点：

已复制

```
输出=模型（**输入）
打印（outputs.last_hidden_state.shape）
```

已复制

```
torch.Size([ 2 , 16 , 768 ])
```



Example

```py
from transformers import AutoTokenizer, AutoModelForCausalLM
import transformers
import torch
import time

tokenizer = AutoTokenizer.from_pretrained(model)
pipeline = transformers.pipeline(
    "text-generation",
    model=model,
    tokenizer=tokenizer,
    torch_dtype=torch.bfloat16,
    trust_remote_code=True,
    device_map="auto",
)


start_time = time.time()
print(start_time)
sequences = pipeline(
    text,
    max_length=200,
    do_sample=True,
    top_k=10,
    num_return_sequences=1,
    pad_token_id=tokenizer.eos_token_id,
)


end_time = time.time()
print(end_time)

for seq in sequences:
    print(f"Result: {seq['generated_text']}")
```











# 模型

```py
from transformers import BertModel
# 加载了预训练模型
model = BertModel.from_pretrained("bert-base-cased")
#  保存模型
model.save_pretrained( "directory_on_my_computer" )

```

生成

```sh
ls directory_on_my_computer

config.json pytorch_model.bin
```

[参考](https://huggingface.co/learn/nlp-course/chapter2/6?fw=pt)

````py
from transformers import AutoTokenizer

checkpoint = "distilbert-base-uncased-finetuned-sst-2-english"
tokenizer = AutoTokenizer.from_pretrained(checkpoint)

sequences = ["I've been waiting for a HuggingFace course my whole life.", "So have I!"]

```
# Returns PyTorch tensors
model_inputs = tokenizer(sequences, padding=True, return_tensors="pt")
# Will truncate the sequences that are longer than the specified max length
model_inputs = tokenizer(sequences, max_length=8, truncation=True)
```


tokens = tokenizer(sequences, padding=True, truncation=True, return_tensors="pt")
output = model(**tokens)
````









# 分词

分词器是 NLP 流水线的核心组件之一。它们服务于一个目的：将文本转换为模型可以处理的数据。模型只能处理数字，因此分词器需要将我们的文本输入转换为数字数据。

```py
from transformers import BertTokenizer

tokenizer = BertTokenizer.from_pretrained("bert-base-cased")
tokenizer("Using a Transformer network is simple")
{'input_ids': [101, 7993, 170, 11303, 1200, 2443, 1110, 3014, 102],
 'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0],
 'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1]}


tokenizer.save_pretrained("directory_on_my_computer")
```

文本转换为数字称为*编码*。编码分两步完成：标记化tokenization，然后转换为输入 ID

- 文本拆分为单词（或单词的一部分、标点符号等），通常称为*token*
- 这些标记转换成数字,它们构建一个张量并将它们提供给模型,tokenizer 有一个*vocabulary*，这就是为什么我们需要使用模型名称实例化分词器，以确保我们使用与预训练模型时相同的规则。

```py
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")

sequence = "Using a Transformer network is simple"
# tokenization
tokens = tokenizer.tokenize(sequence)

print(tokens)

# From tokens to input IDs
ids = tokenizer.convert_tokens_to_ids(tokens)

print(ids)

# decode
decoded_string = tokenizer.decode([7993, 170, 11303, 1200, 2443, 1110, 3014])
print(decoded_string)
```



# Train

[ ](https://huggingface.co/learn/nlp-course/chapter3/4?fw=pt)

```py
from datasets import load_dataset

raw_datasets = load_dataset("glue", "mrpc")
raw_datasets

checkpoint = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(checkpoint)
tokenized_sentences_1 = tokenizer(raw_datasets["train"]["sentence1"])
tokenized_sentences_2 = tokenizer(raw_datasets["train"]["sentence2"])
inputs = tokenizer("This is the first sentence.", "This is the second one.")
inputs


# token_type_ids 用来区分input 哪部分是第一句，第二句
{ 
  'input_ids': [101, 2023, 2003, 1996, 2034, 6251, 1012, 102, 2023, 2003, 1996, 2117, 2028, 1012, 102],
  'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1],
  'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
}

'''
tokenized_dataset = tokenizer(
    raw_datasets["train"]["sentence1"],
    raw_datasets["train"]["sentence2"],
    padding=True,
    truncation=True,
)

等效
'''


def tokenize_function(example):
    return tokenizer(example["sentence1"], example["sentence2"], truncation=True)
# map对数据集的每个元素应用一个函数来工作  batched=True在调用中使用map以便该函数一次应用于我们数据集的多个元素，而不是分别应用于每个元素。这允许更快的预处理
tokenized_datasets = raw_datasets.map(tokenize_function, batched=True)
tokenized_datasets

# 通过DataCollatorWithPadding. 当您实例化它时，它需要一个标记器（以了解要使用哪个填充标记，以及模型是否希望填充位于输入的左侧或右侧）并将完成您需要的一切
data_collator = DataCollatorWithPadding(tokenizer=tokenizer)



# Trainer是定义一个类，该类将包含将用于训练和评估的TrainingArguments所有超参数。Trainer您必须提供的唯一参数是保存训练模型的目录，以及沿途的检查点。对于所有其余部分，您可以保留默认值，这对于基本的微调应该非常有效。

from transformers import TrainingArguments

training_args = TrainingArguments("test-trainer")



from transformers import Trainer

trainer = Trainer(
    model,
    training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["validation"],
    data_collator=data_collator,
    tokenizer=tokenizer,
)
# 使用训练集微调
trainer.train()
```

