---
title: "Named Entity Recognition 相关概念与技术"
date: 2024-07-05
aliases: ["/NLP"]
tags: ["NLP", "NER", "KnowledgeGraph"]
categories: ["NLP"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "命名实体识别快速入门"
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
searchHidden: false
---

## 基本概念

1. **实体**：通常指文本中具有特定意义或指代性强的词或短语，如人名、地名、机构名等。
2. **边界识别**：确定实体在文本中的起始和结束位置。
3. **分类**：将识别出的实体归类到预定义的类别，如人名、地名、组织名、时间表达式、数量、货币值、百分比等。
4. **序列标注**：NER任务中常用的一种方法，将文本中的每个词标注为实体的一部分或非实体。
5. **特征提取**：从文本中提取有助于实体识别的特征，如词性、上下文信息等。
6. **评估指标**：用于衡量NER系统性能的指标，常见的有精确率（Precision）、召回率（Recall）和F1分数。



## 分类命名方法

1. **BIO**：这是最基本的标注方法，其中"B"代表实体的开始（Begin），"I"代表实体的内部（Inside），而"O"代表非实体（Outside）。
2. **BIOES**：这种方法在BIO的基础上增加了"E"表示实体的结束（End），和"S"表示单独的实体（Single）。
3. **BMES**：这种方法使用"B"表示实体的开始，"M"表示实体的中间（Middle），"E"表示实体的结束，"S"表示单个字符实体。



## NER 的一般过程

1. **数据准备**：收集并标注一个包含目标实体的数据集。这个数据集应该包含足够的示例，以便模型能够学习如何识别和分类实体。
2. **选择模型架构**：选择一个适合任务的模型架构，如基于LSTM的序列模型或者是基于Transformers的预训练模型。
3. **特征工程**：根据需要，进行特征工程，提取有助于实体识别的特征，例如词性标注、上下文嵌入等。
4. **模型训练**：使用标注好的数据集来训练模型。这通常包括定义损失函数、选择优化器、设置学习率和训练周期等。
5. **评估与优化**：在独立的验证集上评估模型性能，使用诸如精确率、召回率和F1分数等指标，并根据结果进行模型调优。



## 一个小例子

以当前计组KG为例。

### 数据集

数据格式见 [transformers/examples/pytorch/token-classification at main · huggingface/transformers (github.com)](https://github.com/huggingface/transformers/tree/main/examples/pytorch/token-classification)

数据来源：

- 通过百度百科爬虫 [BaiduSpider/BaiduSpider: BaiduSpider，一个爬取百度搜索结果的爬虫，目前支持百度网页搜索，百度图片搜索，百度知道搜索，百度视频搜索，百度资讯搜索，百度文库搜索，百度经验搜索和百度百科搜索。 (github.com)](https://github.com/BaiduSpider/BaiduSpider) 爬取计算机组合原理的相关术语
- 从计组教材中提取出文本

数据处理：

- 去掉无用字符、HTML标签等无关信息
- 使用Tokenizer将文本数据分解成Token
- 根据Token创建词汇表，每个唯一的Token对应一个唯一的索引
- 将文本中的Token转换为对应的索引值，以便模型能够处理
- 添加位置编码，以便模型能够理解Token在序列中的位置

数据集划分：

- 将数据集划分为训练集、验证集和测试集

格式化数据集：

- 使数据集符合transformer库NER任务模型的输入格式

### 模型

选用中文NER预训练模型：[ckiplab/albert-base-chinese-ner · Hugging Face](https://huggingface.co/ckiplab/albert-base-chinese-ner?text=我叫萨拉，我住在伦敦。)

选用peft框架微调：[peft/examples/token_classification at main · huggingface/peft (github.com)](https://github.com/huggingface/peft/tree/main/examples/token_classification)

### 训练和测试

一般过程，略。

### 应用

可以应用在语料中识别实体了。

### 关系抽取

NER的下一步就是关系抽取了
