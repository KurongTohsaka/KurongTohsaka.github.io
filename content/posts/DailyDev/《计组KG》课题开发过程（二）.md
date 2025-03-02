---
title: "《计组KG》课题开发过程（二）"
date: 2024-09-14
aliases: ["Daily Dev"]
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
searchHidden: false
---

## 前言

自从上次记录已经过去了一个月，整个课题进展不大。原因一个是暑期有点摆，另一个是关系抽取确实比较繁琐。不管怎么说，来记录下吧。



## NER 数据集下的模型训练

首先需要声明的是该阶段的模型不参与于最后 KG 的构建，目的仅仅是跑通模型训练、验证的过程，为后续阶段提供便利。

### NER 数据集

**该部分信息在完成 RE 部分后可能会发生些微变动，仅作参考，后续会做调整。**

共标记5147条中文语句，实体共标注1472个。

下面是各个标签下的数量统计：

| **Label** | **Count** |
| --------- | --------- |
| TECH      | 388       |
| COMP      | 382       |
| STOR      | 170       |
| DATA      | 133       |
| INST      | 105       |
| ARCH      | 71        |
| IO        | 61        |
| PERF      | 54        |
| PROG      | 52        |
| CORP      | 17        |
| ALG       | 16        |
| PROT      | 15        |
| PER       | 4         |
| GRP       | 4         |

### 模型选择

模型有两大类：

- 传统深度学习方法
  - CNN-CRF
  - BiLSTM-CRF
- BERT 系预训练模型，输出层为 CRF 或 MLP+Softmax
  - BERT：BERT 是一个双向 Transformer 模型，通过掩码语言模型（Masked Language Model, MLM）和下一句预测（Next Sentence Prediction, NSP）任务进行预训练
  - RoBERTa：RoBERTa 是对 BERT 的优化版本，移除了 NSP 任务，并采用了动态掩码策略
  - ALBERT：ALBERT 是 BERT 的轻量级版本，通过参数共享和嵌入参数因子化来减少模型大小
  - XLM-RoBERTa：XLM-RoBERTa 是针对多语言的预训练模型，基于 RoBERTa 和 XLM 的结合

这里选择的是 XLM-RoBERTa，预训练模型选择的是 [FacebookAI/xlm-roberta-large-finetuned-conll03-english · Hugging Face](https://huggingface.co/FacebookAI/xlm-roberta-large-finetuned-conll03-english) 

模型代码如下：

```python
class XLMRobertaForNER(XLMRobertaPreTrainedModel):
    def __init__(self, config, use_bilstm=False, use_crf=True):
        super(XLMRobertaForNER, self).__init__(config)
        self.use_bilstm = use_bilstm
        self.use_crf = use_crf
        self.xlm_robert = XLMRobertaModel(config, add_pooling_layer=False)
        self.dropout = nn.Dropout(0.1)
        self.cls = nn.Linear(config.hidden_size, config.num_labels)
        if self.use_bilstm:
            self.bilstm = nn.LSTM(
                input_size=config.hidden_size,
                hidden_size=config.hidden_size,
                num_layers=1,
                batch_first=True,
                bidirectional=True
            )
        if self.use_crf:
          	self.crf = CRF(config.num_labels, batch_first=True)
        else:
          	self.softmax = nn.Softmax(dim=1)

        self.init_weights()

    def forward(self, input_ids, attention_mask=None, labels=None):
        outputs = self.xlm_robert(input_ids, attention_mask=attention_mask)
        sequence_output = outputs[0]

        if self.use_bilstm:
            bilstm_output, _ = self.bilstm(sequence_output)
            sequence_output = torch.cat((sequence_output, bilstm_output), dim=2)  # 合并BiLSTM的输出和初始输入

        sequence_output = self.dropout(sequence_output)
        emissions = self.cls(sequence_output)

        if self.use_crf:
            if labels is not None:
                labels = torch.where(labels==-100, torch.tensor(0).to(labels.device), labels)
                loss = -self.crf(emissions, labels, mask=attention_mask.byte(), reduction='mean')
                return loss
            else:
              	prediction = self.crf.decode(emissions, mask=attention_mask.byte())
              	return prediction
        else:
            logits = self.softmax(emissions)
            return logits
```

### 训练过程

1. 初始化 DataLoader、Tokenizer ；
2. 修改 Model Config、Training Params ；
3. 初始化 Model，启动训练 Loop 。

以下是本次训练所选取的各项参数：

```python
training_params = {
    'model_name': xlm_roberta_model_name,
    'weight_decay': 1e-2,
    'warmup_steps': 500,
    'warmup_proportion': 0.1,
    'learning_rate': 5e-5,
    'adam_beta1': 0.9,
    'adam_beta2': 0.999,
    'adam_epsilon': 1e-8,
    'num_train_epochs': 16,
}
```

Loss 函数的计算方式在代码中：

```python
loss = -self.crf(emissions, labels, mask=attention_mask.byte(), reduction='mean')
```

Optimizer 选择的是 AdamW，同时运用了 Learning Rate Scheduler 来完成暖机。

### 验证结果

**关于这部分，数据不保证准确、全面，因此只作为参考，切勿用于其他用途。**

只使用1000条小批次 NER 训练集，经过16轮训练后，各 base 模型在 NER 测试集上的泛化性能：

| Model               | Precision | Recall    | F1        |
| ------------------- | --------- | --------- | --------- |
| BERT-CRF            | 74.1%     | 81.8%     | 74.6%     |
| ALBERT-CRF          | 70.7%     | 73.4%     | 68.6%     |
| **XLM-RoBERTa-CRF** | **88.1%** | **95.4%** | **90.3%** |

各项数据都是 XLM-RoBERTa-CRF 大幅领先。

> XLM-RoBERTa-CRF 所选取的预训练模型经过了 Facebook 团队在超过 2.5TB 的多语言大规模数据集上的训练，模型参数达 0.56B 。

关于这部分数据目前来看缺陷相当大：

- 数据不全：无法反映数据集的完整信息，模型也更容易出现过拟合、测试集泛化性能一般
- 模型选择不具有代表性：只有 BERT 系，没有其他结构类型的模型，如 CNN、BiLSTM 等
- 没有做消融实验：没有 Softmax、CRF 层的对比
- 性能指标选取以及模型性能差距过大：在做实验时，所选取的这些指标都是 Samples 上的，即整个数据集上的指标，不能完整反应模型性能。而且这些模型的性能差距不是在5个点内，都10个点往上了，所以仅作参考是没啥问题的



## RE 数据集的构建

### 关系定义

| 关系名称 | 英文名称  | 说明                                         |
| -------- | --------- | -------------------------------------------- |
| 包含     | contain   | A 包含 B 的内容（单向）                      |
| 顺序     | sequence  | 学习 A 前需先学 B，或学习 A 后支持 B（单向） |
| 同义     | synonymy  | 名称不同但指同一内容（双向）                 |
| 相关     | related   | A 与 B 存在相关性（双向）                    |
| 特性     | attribute | 包括功能、性能、方法等特点（单向）           |

### 预标注前的准备工作

**该部分信息在进行预标注过程中可能会发生变动，仅作参考，后续会做调整。**

对302条文本进行了关系标注，其中184条用于训练、60条用于验证、58条用于测试。

下面是一个简单的统计：

| **Contain** | **Related** | **Attribute** | **Synonymy** | **Sequence** |
| ----------- | ----------- | ------------- | ------------ | ------------ |
| 188         | 183         | 158           | 82           | 84           |

### 预标注

模型选择的 BiLSTM-MAML 。

> **MAML**（**Model-Agnostic Meta-Learning**）：是一种元学习算法，同样是一种 Few-Shot Learning 算法。核心思想是通过学习模型参数的初始值，使得模型能够在面对新任务时快速适应。

MAML 算法在使得模型快速适应的同时，也增加了训练阶段的计算量，这会导致训练时间的延长，不利于后续模型验证。所以决定选用结构更简单、同时性能较好的 BiLSTM 模型。



（模型的训练、标注效果）



## NER-RE 数据集的构建

（暂定）



## KG 的展示

（暂定）

