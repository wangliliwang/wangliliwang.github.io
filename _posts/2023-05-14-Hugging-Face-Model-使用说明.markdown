---
layout: post
title:  "🤗 Hugging Face Model 使用说明"
date:   2023-05-07 13:11:49 +0800
categories: AI
tags: ["AI"]
toc: true
language: chinese
sidebar: true
author: Wang Li
---

本文用于不了解transformer model的读者入门使用。
主要内容用 transformers 这个 Python 包，说明 model 如何产生、使用，以及 Hugging Face model repo 中的文件。

## Model 是什么？

model 是一个可以处理输入数据的函数，输入数据可以是图片、文本、音频等，输出数据可以是分类、回归、文本生成等。
model 的参数是在训练阶段学习到的，这些参数可以被保存，以便在推理（调用）阶段使用。

### 空白 model

空白的 model 只包含了模型的结构，参数是随机生成的，输出也是随机的，没有任何实际用途。

下面是加载一个空白的 model 的代码示例：
```python
from transformers import BertConfig, TFBertModel

# Building the config
config = BertConfig()

# Building the model from the config
model = TFBertModel(config)
```

### pretrained model

pretrained model 是在空白 model 上训练出来的，对于目前流行的 LLM，需要使用大量的训练新数据，消耗大量的算力、时间（数周以上）。

此阶段的训练

## Model 处理流程

下面是一个NLP Text Classification任务：

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-distilbert-base-uncased-finetuned-sst-2-english.hf.space"></gradio-app>

我们的输入是一行文本（如："I love you."），输出是一个分类结果，包含了预测的类别（Positive/Negative）和概率。

对于其他任务，虽然输入输出不同，但是处理流程是一样的。如下图：
（处理流程图）

### pre-processing

因为 model 不能直接处理 text ，所以需要将文本转换成数字，这个过程叫做tokenize（pre-processing中只包含这一步），tokenize 的过程是由 tokenizer 完成的。主要做的事情如下：
1. 将文本切分为word、subword、标点标点符号等，这些统称为 token
2. 将 token 映射为数字
3. 添加一些对模型有帮助的额外输入

获取 text 的 token 的代码示例及输出如下：
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

### model forward

模型接受 tokens 作为输入，输出 hidden states（也称为 features），此结果需要经过 head 层，才能转化为分类及分数。
model 的处理细节如下图：
（todo 另一个处理图）

示例代码（model+head处理）如下：
```python
from transformers import TFAutoModelForSequenceClassification

checkpoint = "distilbert-base-uncased-finetuned-sst-2-english"
# 下面的方法会找到预训练阶段的 model 及 head，不同的任务对应不同的类
model = TFAutoModelForSequenceClassification.from_pretrained(checkpoint)
outputs = model(inputs)
print(outputs.logits)

# 输出如下（以第一个元素[-1.5606991,  1.6122842]为例，-1.5606991 表示类别，1.6122842 表示分数）：
# <tf.Tensor: shape=(2, 2), dtype=float32, numpy=
#     array([[-1.5606991,  1.6122842],
#            [ 4.169231 , -3.3464472]], dtype=float32)>
```

### post-processing

model 产生的结果，并不能被直接用来展示给用户，还需要经过一层处理，示例代码如下：
```python
import tensorflow as tf

predictions = tf.math.softmax(outputs.logits, axis=-1)
print(predictions)

# 输出如下（以第一个元素[0.04019517, 0.95980483]为例，0.04019517 表示类别，0.95980483 表示概率）：
# tf.Tensor(
# [[4.01951671e-02 9.59804833e-01]
# [9.9945587e-01 5.4418424e-04]], shape=(2, 2), dtype=float32)
```

类别的 label 存储在 model repo 的 `configs.json` 的 `id2label` 字段中，示例代码模型此字段的值为 `{0: 'NEGATIVE', 1: 'POSITIVE'}`，所以最终输出为：
1. 第一个句子: NEGATIVE: 0.0402, POSITIVE: 0.9598
2. 第二个句子: NEGATIVE: 0.9995, POSITIVE: 0.0005

### 使用 pipeline

transformers 的 `pipeline` 方法，可以将上面三个步骤聚合起来，更方面使用模型，使用示例如下：
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

## 参考

1. [Hugging Face Docs](https://huggingface.co/docs)
