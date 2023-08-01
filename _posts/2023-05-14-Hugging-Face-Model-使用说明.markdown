---
layout: post
title:  "🤗 Hugging Face Model 使用说明"
date:   2023-05-14 14:11:49 +0800
categories: Computer-Science
tags: ["AI"]
toc: true
language: chinese
sidebar: true
author: Wang Li
---

本文的主要内容用 `transformers` 这个 Python 包，说明 model 如何产生、使用，以及 Hugging Face model repo 中的文件。
本文的 model 指的是 transformer 架构的 model，此类 model 通常用于 NLP 任务，如文本分类、文本生成等。

## Model 是什么？

model 是一个可以处理输入数据的函数，输入数据可以是图片、文本、音频等，输出数据可以是分类、回归、文本生成等。
model 的参数值是在训练阶段学习到的，这些参数可以被保存，以便在后续阶段使用。

有证据表明，训练数据量越大、model 参数量越大，效果越好，所以现在最流行的 model 是 LLM（Large Language Model）。所以当前 model 的产生为两个阶段：
1. pre-train：使用大量的数据，进行自监督训练。这样训练出来的 model，对特定任务帮助不大。但是可以在下一个阶段被 transfer 成特定任务的 model。 
2. transfer：使用针对特定任务的数据（数据规模一般较 pre-train 阶段小得多）训练上一阶段产生的 model，这一阶段是监督训练。此阶段也称为 fine-tune。

之所以分成两个阶段，而不是直接针对特定任务训练，正是因为上面的设计可以复用需要大量数据训练的 pre-train 阶段。

### 使用空白 model

空白的 model 只包含了模型的结构，参数是随机生成的，输出也是随机的，没有任何实际用途。

下面是加载一个空白的 model 的代码示例：
```python
from transformers import BertConfig, TFBertModel

# Building the config
config = BertConfig()

# Building the model from the config
model = TFBertModel(config)
```

### 使用非空白 model

使用 pre-trained model 以及 transferred model 的代码示例如下：
```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")

# 后续就可以使用 classifier 处理任务了
```

从 model card 可以找到 model 的详细信息，包括类型。

### 保存 model

使用如下代码，可以将 model 保存到本地：
```python
model.save_pretrained("directory_on_my_computer")
```

此步骤会生成两个文件，架构描述 `config.json` 与 model 参数值，后者的文件名随模型类型变化。详情见"Model Repo 中都有什么"一节。 

## 任务处理流程

下面是一个 NLP Text Classification 任务：

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-distilbert-base-uncased-finetuned-sst-2-english.hf.space"></gradio-app>

我们的输入是一行文本（如："I love you."），输出是一个分类结果，包含了预测的类别（Positive/Negative）和概率。

对于其他任务，虽然输入输出不同，但是处理流程是一样的。
![](/assets/image/20230514-huggingface-transformer/full_nlp_pipeline.svg)

### pre-processing

因为 model 不能直接处理 text ，所以需要将文本转换成数字，这个过程叫做 tokenize（pre-processing中只包含这一步），tokenize 的过程是由 tokenizer 完成的。主要做的事情如下：
1. 将文本切分为 word、subword、标点符号等，这些统称为 token
2. 将 token 映射为数字
3. 添加一些对模型有帮助的额外输入

获取 text 的 token 的代码及输出示例：
```python
from transformers import AutoTokenizer

checkpoint = "distilbert-base-uncased-finetuned-sst-2-english"
# 下面的方法会自动寻找此模型预训练时的 tokenizer
tokenizer = AutoTokenizer.from_pretrained(checkpoint)

raw_inputs = [
    "I've been waiting for a HuggingFace course my whole life.",
    "I hate this so much!",
]
inputs = tokenizer(raw_inputs, padding=True, truncation=True, return_tensors="tf")
print(inputs)

# 输出如下（只关注 input_ids 即可）
# {
#     'input_ids': <tf.Tensor: shape=(2, 16), dtype=int32, numpy=
#         array([
#             [  101,  1045,  1005,  2310,  2042,  3403,  2005,  1037, 17662, 12172,  2607,  2026,  2878,  2166,  1012,   102],
#             [  101,  1045,  5223,  2023,  2061,  2172,   999,   102,     0,     0,     0,     0,     0,     0,     0,     0]
#         ], dtype=int32)>, 
#     'attention_mask': <tf.Tensor: shape=(2, 16), dtype=int32, numpy=
#         array([
#             [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
#             [1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0]
#         ], dtype=int32)>
# }
```

### model-processing

model 接受 tokens 作为输入，输出 hidden states（也称为 features），此结果需要经过 head 层，才能转化为分类及分数。
model 的处理细节如下：
![](/assets/image/20230514-huggingface-transformer/transformer_and_head.svg)

示例代码（model+head 处理）如下：
```python
from transformers import TFAutoModelForSequenceClassification

checkpoint = "distilbert-base-uncased-finetuned-sst-2-english"
# 下面的方法会找到预训练阶段的 model 及 head，不同的任务对应不同的类
model = TFAutoModelForSequenceClassification.from_pretrained(checkpoint)
outputs = model(inputs)
print(outputs.logits)

# 输出如下（以第一个元素[-1.5606991,  1.6122842]为例，-1.5606991 是类别1的分数，1.6122842 是类别2的分数）
# <tf.Tensor: shape=(2, 2), dtype=float32, numpy=
#     array([[-1.5606991,  1.6122842],
#            [ 4.169231 , -3.3464472]], dtype=float32)>
```

### post-processing

model 产生的结果，用户并不能直接处理，还需要经过一层处理，示例代码如下：
```python
import tensorflow as tf

predictions = tf.math.softmax(outputs.logits, axis=-1)
print(predictions)

# 输出如下（以第一个元素[0.04019517, 0.95980483]为例，0.04019517 是类别1的概率，0.95980483 是类别2的概率）
# tf.Tensor(
# [[4.01951671e-02 9.59804833e-01]
# [9.9945587e-01 5.4418424e-04]], shape=(2, 2), dtype=float32)
```

model label 存储在 model repo 的 `config.json` 的 `id2label` 字段中，示例代码模型此字段的值为 `{0: 'NEGATIVE', 1: 'POSITIVE'}`，所以最终输出为：
1. 第一个句子: `NEGATIVE: 0.0402, POSITIVE: 0.9598`
2. 第二个句子: `NEGATIVE: 0.9995, POSITIVE: 0.0005`

### 使用 pipeline

transformers 的 `pipeline` 方法，可以将上面三个步骤聚合起来，更方面用户使用，示例如下：
```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")
outputs = classifier(
    [
        "I've been waiting for a HuggingFace course my whole life.",
        "I hate this so much!",
    ]
)

# 输出如下：
# [{'label': 'POSITIVE', 'score': 0.9598047137260437},
#  {'label': 'NEGATIVE', 'score': 0.9994558095932007}]
```

pipeline 传入的参数是任务类型，细节可以自行查阅文档。

## Model Repo 中都有什么

model 中主要包含三类文件：
1. `config.json`: model 的架构配置，比如 model 的层数、hidden size、dropout 等
2. `pytorch_model.bin` / `tf_model.h5` 等: model 的权重，也就是 model 的参数值。不同的框架生成的文件不同。
3. `README.md`: model 的介绍，包括 model 的训练数据、训练参数、训练结果等，model card 是根据此文件生成的

## 参考

1. [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course/chapter0/1?fw=pt)
