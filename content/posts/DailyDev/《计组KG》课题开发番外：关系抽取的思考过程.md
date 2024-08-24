---
title: "《计组KG》课题开发番外：关系抽取的思考过程"
date: 2024-08-24
aliases: ["/Daily Dev"]
tags: ["Daily Dev", "Project Dev"]
categories: ["Daily Dev"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: ""
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## 现状

目前课题进行到了关系抽取这一步。在看完之前的RE综述后，我决定整一个 Joint 的 NER+RE 模型。目前已经完成了 NER 的标注工作，下一步工作自然就是 RE 的标注。但是 RE 的标注要比 NER 要困难的多，一个是对于关系的定义依赖于实体类型、句子词性等多种复杂要素，二是手工标注势必工作量巨大。所以 RE 的标注有什么解决方法呢？

我决定把整个思考过程记录下来，方便复盘。



## 相关开源库

[thunlp/OpenNRE：用于神经关系提取 （NRE） 的开源包 (github.com)](https://github.com/thunlp/OpenNRE)

[SapienzaNLP/relik：检索、读取和 LinK：学术预算上的快速准确的实体链接和关系提取 （ACL 2024） (github.com)](https://github.com/SapienzaNLP/relik?tab=readme-ov-file)

[huggingface/setfit：使用句子转换器进行高效的少样本学习 (github.com)](https://github.com/huggingface/setfit)

[EleutherAI/lm-evaluation-harness：一种用于语言模型小样本评估的框架。 (github.com)](https://github.com/EleutherAI/lm-evaluation-harness)



## 关系抽取的可能方案

首先能想到的两种常见方案：

- 使用 RE 预训练模型预标注：和 NER 的标注工作流程一致，但是中文 RE 预训练模型很难找
- 远程监督：使用已有的知识库或实体关系词典，对大规模文本进行远程监督标注。但是不适合当前课题的小规模数据集，而且没有现成的大量 RE 标注数据

以上两种方案中只有预训练模型的预标注还算可行，那有没有什么方法可以加速这一过程？可以看看下面的两种方法：

- 半监督学习：结合少量人工标注数据和大量未标注数据，通过半监督学习的方法训练模型

- few-shot 小样本学习：使用少量人工标注的数据对few-shot模型进行训练，以提高模型在少样本情况下的泛化能力

这两种方案值得单独进行介绍。

### 半监督学习

半监督学习（Semi-Supervised Learning）是一种结合了监督学习和无监督学习的机器学习方法。它利用少量的标记数据和大量的未标记数据来训练模型，从而提高模型的泛化能力和性能。

半监督学习的样本标注依赖假设，以下是部分常见假设：

- 平滑性假设：如果两个数据点在高密度区域中且距离很近，那么它们的输出也应该相似
- 聚类假设：如果数据点形成簇，那么同一簇中的数据点应该属于同一类
- 流形假设：高维数据通常位于低维流形上，同一流形上的数据点具有相同的标签

常用的方法有：

- 一致性正则化：假设对未标记数据加入小扰动后，其分类结果不应改变
- 伪标签：使用已标记数据训练初始模型，然后用该模型对未标记数据进行预测，生成伪标签，再将这些伪标签数据加入训练集中进行再训练
- 生成式模型：利用生成模型（如GANs）从数据分布中生成样本，并将这些样本用于训练分类器

优势：

- 利用未标注数据，提高模型的泛化能力和性能
- 较低标注成本
- 适应性强，可以应用于多种任务

缺点：

- 依赖数据假设：半监督学习通常假设未标记数据和标记数据在特征空间中具有相似性，这在实际应用中并不总是成立。如果这些假设不成立，可能会导致模型性能下降
- 标签传播误差
- 数据不平衡问题

我曾经在 kaggle 比赛中使用过半监督学习中的伪标签方法，只从工程实现的角度看比较容易，但是模型性能不一定有提升。

该方法在本任务上的应用比较困难：

- 使用这种方法如何生成比较准确的标签是一个比较难的问题，尤其是现在还没有标签
- 不好与 NER 构建 Joint 模型

### few-shot 小样本学习

few-shot 学习是一种机器学习方法，旨在通过极少量的训练样本进行学习和泛化。与传统的机器学习方法需要大量标注数据不同，few-shot 学习能够在数据稀缺的情况下仍然有效地进行学习和预测。

few-shot 主要分为以下几类：

- 基于生成模型的方法：通过生成模型来模拟数据分布，从而生成更多的训练样本

- 基于判别模型的方法：通过判别模型直接学习样本之间的相似性或差异性

- 元学习（Meta-learning）：通过学习如何学习，使模型能够快速适应新任务

优势：

- 数据需求少：能够在少量标注数据的情况下进行有效学习，降低数据标注成本
- 快速适应新任务：通过元学习等方法，模型可以快速适应新的任务和数据分布

缺点：

- 对样本质量要求高：由于样本数量少，样本的质量对模型性能影响较大。
- 模型复杂度高：一些 few-shot 学习方法（如元学习）需要复杂的模型设计和训练过程

在 RE 中使用 few-shot 的一种基于预标注的方案：

- 先少量标注下每个关系标签下的样本，然后用这部分数据去微调一个结合了 few-shot 的小预训练模型，再用该模型去做预标注。得到预标注数据集之后，再去训练一个更大的预训练模型来完成 RE



## 决定采用的方案

我决定用 few-shot 方案，原因有以下几点：

1. KG 对关系抽取任务的指标要求很高，使用半监督学习的话会使数据集中满是噪声和错误样本
2. 标注少量数据，使用元学习来完成 few-shot 学习。这一过程很好的与预标注契合到了一起
3. few-shot 近几年相对热门，蹭蹭热度

元学习参见：

- [小样本学习——概念、原理与方法简介（Few-shot learning） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/258562899)

目前暂定的可行方案，大致如下：

- 少量标注下每个关系标签下的样本
- 训练一个多层 BiLSTM-CRF 模型作为 few-shot 学习的 base ，然后在训练阶段用元学习中的 MAML 方法优化训练过程
- 用上面的模型做一次预标注，然后再人工优化一下
- 构建一个 NER-RE 的 Joint End2end模型

示例代码参考如下：

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchcrf import CRF

# 定义BiLSTM-CRF模型
class BiLSTM_CRF(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_dim, tagset_size):
        super(BiLSTM_CRF, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_dim // 2, num_layers=2, bidirectional=True, batch_first=True)
        self.hidden2tag = nn.Linear(hidden_dim, tagset_size)
        self.crf = CRF(tagset_size, batch_first=True)

    def forward(self, x):
        embeds = self.embedding(x)
        lstm_out, _ = self.lstm(embeds)
        emissions = self.hidden2tag(lstm_out)
        return emissions

    def loss(self, emissions, tags, mask):
        return -self.crf(emissions, tags, mask=mask)

    def decode(self, emissions, mask):
        return self.crf.decode(emissions, mask=mask)

# MAML训练函数
def train_maml(model, tasks, inner_lr, outer_lr, inner_steps, outer_steps):
    optimizer = optim.Adam(model.parameters(), lr=outer_lr)
    for step in range(outer_steps):
        meta_loss = 0
        for task in tasks:
            # 内层优化
            task_model = BiLSTM_CRF(vocab_size, embedding_dim, hidden_dim, tagset_size)
            task_model.load_state_dict(model.state_dict())
            task_optimizer = optim.SGD(task_model.parameters(), lr=inner_lr)
            for _ in range(inner_steps):
                x, y, mask = task.sample()
                emissions = task_model(x)
                loss = task_model.loss(emissions, y, mask)
                task_optimizer.zero_grad()
                loss.backward()
                task_optimizer.step()
            
            # 外层优化
            x, y, mask = task.sample()
            emissions = task_model(x)
            loss = task_model.loss(emissions, y, mask)
            meta_loss += loss
        
        optimizer.zero_grad()
        meta_loss.backward()
        optimizer.step()

# 示例任务
class Task:
    def __init__(self, data):
        self.data = data

    def sample(self):
        x, y, mask = zip(*self.data)
        return torch.tensor(x), torch.tensor(y), torch.tensor(mask)

# 训练MAML模型
vocab_size = 5000  # 词汇表大小
embedding_dim = 100  # 词向量维度
hidden_dim = 256  # 隐藏层维度
tagset_size = 10  # 标签集大小

model = BiLSTM_CRF(vocab_size, embedding_dim, hidden_dim, tagset_size)
tasks = [Task(data) for data in task_data]  # task_data为预处理好的任务数据
train_maml(model, tasks, inner_lr=0.01, outer_lr=0.001, inner_steps=5, outer_steps=1000)

```

