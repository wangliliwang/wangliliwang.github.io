---
layout: post
title:  "🤗 Hugging Face 简明教程"
date:   2023-05-07 13:11:49 +0800
categories: AI
tags: ["AI"]
toc: true
language: chinese
sidebar: true
author: Wang Li
---

本文用于不了解 Hugging Face 的读者入门使用。

介绍：[Hugging Face](https://huggingface.co/) 是机器学习界的 GitHub。

Hugging Face 支持三种 Repo：
1. Model：机器学习模型。
2. Dataset：数据集。此种 Repo 功能比较简单，本文不做详述。
3. Space：Demo APP。

![](/assets/image/20230507-huggingface/1-top-menu.png)

Hugging Face 的典型使用场景：
1. 寻找、分享 Model/Dataset/代码/Model Demo。
![](/assets/image/20230507-huggingface/2-repo-models.png)
2. 试用 Model。网站上的 Model 会自动生成可供用户使用的 Demo。
![](/assets/image/20230507-huggingface/10-model-widget.png)
3. 定制 Model Demo，并免费发布到 Hugging Face。花十分钟时间即可制作一个图片归类/翻译/Chat等 Demo。如下：

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-image-classification.hf.space"></gradio-app>

此外，Hugging Face 支持 [Chat](https://huggingface.co/chat/)（对标 ChatGPT）、[AutoTrain](https://huggingface.co/autotrain)（在线训练 Model、根据数据集自动挑选合适的 Model）、[Inference Endpoints](https://huggingface.co/inference-endpoints)（生产环境部署 Model 的全套解决方案）等功能，有兴趣的读者可自行查阅文档。

下文详细描述 Model/Space 两种 Repo 的使用。

## Repo 基础

### 新建/查看 Repo

在 Hugging Face 的首页，点击左上角的 New 按钮，即可新建三种 Repo，如下图：

![](/assets/image/20230507-huggingface/4-repo-new.png)

点击右上角自己的头像，进入 Profile 页面，即可查看自己的 Repo：

![](/assets/image/20230507-huggingface/5-repo-mine.png)

### Model/Dataset Card

每个 Repo 的 README.md 会被渲染为 Card 放在 Repo 首页展示，所以书写 README.md 时需要遵循一定的格式，并带上必要的信息。此文件分为两部分，上面的部分是以两个`---`包围的 yaml 格式的 metadata，剩下的部分是非格式化的描述。

metadata 会在很多场景下被用到。比如，网站的搜索功能会部分依赖 metadata 的 tags 参数。

下面两张图是 [bert-base-uncased](https://huggingface.co/bert-base-uncased) 的 Model Card 以及对应的 README.md 的内容：

![](/assets/image/20230507-huggingface/6-model-card.png)
![](/assets/image/20230507-huggingface/7-readme.png)

### 配置本地环境

需要修改 Repo 文件的读者才需要配置下面的东西。只体验 Demo 则无需配置。

Step 1: 使用 Git 作为版本控制工具。

Step 2: Model/Dataset 占用空间较大，需要使用 Git LFS，此时文件不是以 blobs 的方式存储在 Repo 中，而是存储在 LFS Server 中。

```shell
brew install git-lfs

# Setup Git LFS on your system
git lfs install
```

Step 3: 配置 huggingface_hub
```shell
python -m pip install huggingface_hub

# 注：login 时需要的 token 从 https://huggingface.co/settings/tokens 获取
huggingface-cli login
```

## Model Repo

Model 是机器学习训练的成果，它接受用户输入并产出结果。Hugging Face 根据任务类型（text-classification, token-classification, translation 等）、使用的框架、使用的数据集等不同纬度，将 Model 归类，任务类型、使用的框架两种维度的分类标签如下：

![](/assets/image/20230507-huggingface/8-model-tasks.png)
![](/assets/image/20230507-huggingface/9-model-libs.png)

有两种试用 Model 的方法，说明如下：

### 试用方法1：Widget

此网站提供的 Widget 功能可以让用户快速试用 Model。以 [bert-base-uncased](https://huggingface.co/bert-base-uncased) 模型为例，在右边的输入框填写文本，点击 `Compute` 即可看到结果：

![](/assets/image/20230507-huggingface/10-model-widget.png)

Widget 类型与任务类型挂钩。Hugging Face 优先从 metadata pipeline_tag 字段中读取任务类型，其次从 tags 或者 configs.json 中匹配任务类型。如果读取失败，或者不支持这种任务类型，那么不会展示 Widget。

查看及体验所有支持的任务类型的 Widget，[点击此链接](https://huggingface-widgets.netlify.app/)。

### 试用方法2：Inference API

使用这种方式可以直接发送 HTTP 请求到 Hugging Face，速度比 Widget 方式快 2~10 倍，且后者内部也是通过 Inference API 发出请求的。调用 [bert-base-uncased](https://huggingface.co/bert-base-uncased) Model 的示例如下：

说明：获取 Access Token 的页面：[Access Tokens](https://huggingface.co/settings/tokens)。每个账户有一定的免费调用额度，用量见 [Dashboard](https://api-inference.huggingface.co/dashboard/)。

```shell
curl https://api-inference.huggingface.co/models/bert-base-uncased \
        -X POST \
        -d 'The goal of life is [MASK].' \
        -H "Authorization: Bearer ${Your Access Token}"

# 返回示例
[{
	"score": 0.10933294892311096,
	"token": 2166,
	"token_str": "life",
	"sequence": "the goal of life is life."
}, {
	"score": 0.039418820291757584,
	"token": 7691,
	"token_str": "survival",
	"sequence": "the goal of life is survival."
}, {
	"score": 0.03293056786060333,
	"token": 2293,
	"token_str": "love",
	"sequence": "the goal of life is love."
}, {
	"score": 0.030096111819148064,
	"token": 4071,
	"token_str": "freedom",
	"sequence": "the goal of life is freedom."
}, {
	"score": 0.024967176839709282,
	"token": 17839,
	"token_str": "simplicity",
	"sequence": "the goal of life is simplicity."
}]
```

## Space Repo

介绍：快速创建、部署 Model Demo。

说明：
1. 不仅支持运行 Model 程序，也支持运行其他任意程序，比如输出 `Hello, World!` 的程序。
2. 提供免费的部署、运行资源，配置为 2 vCPU，16GB RAM，50Gb Disk，如果不够用，可以氪金。
![](/assets/image/20230507-huggingface/11-space-hardware.png)
3. 提供 Demo 独立网页入口
4. 支持以内嵌方式使用 Demo 
5. 官方提供四个SDK（对应四个 Space Repo 运行环境），其中 `streamlit`, `gradio` 是 Python SDK，需要用 Python 开发；Docker、Static SDK 则没有必须使用 Python 的限制。
![](/assets/image/20230507-huggingface/12-space-sdk.png)

下文以两个 Demo 为例说明此功能。

特殊说明：目前 Hugging Face 非付费用户过多导致资源紧张，某些时候会出现部署卡住的情况，点击 `Repo Settings - Factory reboot` 多试几次就能部署成功。

### Demo1：输出 `Hello, {input}`

Step 1: 新建一个名为 `hello-input` 的 Space Repo，使用 `gradio` SDK

Step 2: git clone 代码到本地 

Step 2: 增加文件 `app.py`，内容如下：
```python
import gradio as gr

def greet(input):
    return "Hello, " + input + "!"

gr.Interface(
    fn=greet,
    inputs=gr.Textbox(lines=1, placeholder="Name Here..."),
    outputs="text",
    title="Hello, World!",
).launch()
```

Step 3: 本地安装 `gradio`: `pip install gradio`

Step 4: 本地运行 Demo：`gradio app.py`，在浏览器访问 `http://127.0.0.1:7861/`，体验功能

Step 5: git push 代码到仓库，代码会被自动部署，可通过以下三种方式体验 Demo，别人也可以随意使用：

![](/assets/image/20230507-huggingface/13-space-hello-input.png)

1. Demo 主页：`https://huggingface.co/spaces/glrh11/hello-input`
2. Demo 的独立网页入口：`https://glrh11-hello-input.hf.space/`
3. Demo 嵌入示例如下：

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-hello-input.hf.space"></gradio-app>

嵌入代码如下：
```html
<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-hello-input.hf.space"></gradio-app>
```

### Demo2：调用 Model 进行图片分类

Step 1: 新建一个名为 `image-classification` 的 Space Repo，使用 `gradio` SDK

Step 2: 在模型检索页面，选择 `image-classification` 标签，此处我们选择第一个 `microsoft/resnet-50` Model
![](/assets/image/20230507-huggingface/14-space-image-classification.png)

Step 3: 新增 `app.py`，内容如下：

```python
import gradio as gr
from transformers import pipeline

pipeline = pipeline(task="image-classification", model="microsoft/resnet-50")

def predict(image):
    predictions = pipeline(image)
    return {p["label"]: p["score"] for p in predictions}

gr.Interface(
    predict,
    inputs=gr.inputs.Image(label="Upload hot dog candidate", type="filepath"),
    outputs=gr.outputs.Label(num_top_classes=5),
    title="Image Classification",
).launch()
```

Step 4: 新增 `requirements.txt`，内容如下：

```text
transformers
torch
```

Step 5: push 代码到仓库，自动部署后可以通过下面三种方式查看效果：

![](/assets/image/20230507-huggingface/15-space-image-classification.png)

1. Demo 主页：`https://huggingface.co/spaces/glrh11/image-classification`
2. Demo 的独立网页入口：`https://glrh11-image-classification.hf.space/`
3. Demo 嵌入示例如下：

<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-image-classification.hf.space"></gradio-app>

嵌入代码如下：
```html
<script
	type="module"
	src="https://gradio.s3-us-west-2.amazonaws.com/3.28.3/gradio.js"
></script>

<gradio-app src="https://glrh11-image-classification.hf.space"></gradio-app>
```

## 总结

Hugging Face 提供了强大的机器学习 Model 管理、试用能力，是 AI 从业者的利器。

## 参考

1. [Hugging Face Docs](https://huggingface.co/docs)
