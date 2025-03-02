---
title: "《Summarization as Indirect Supervision for Relation Extraction》笔记"
date: 2024-09-29
aliases: ["/PaperNotes"]
tags: ["PaperNotes", "RE", "EMNLP"]
categories: ["PaperNotes"]
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
cover:
    image: "cover/Summarization_as_Indirect_Supervision_for_Relation_Extraction.png" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
searchHidden: false
---

## Link

[[2205.09837v2\] Summarization as Indirect Supervision for Relation Extraction (arxiv.org)](https://arxiv.org/abs/2205.09837v2)

Accepted EMNLP 2022.



## Intro

关系抽取（RE）旨在从文本中提取实体之间的关系。例如，给定句子“Steve Jobs 是 Apple 的创始人”，RE 模型会识别出“创立”这一关系。RE 是自然语言理解的重要任务，也是构建知识库的关键步骤。先进的 RE 模型对于对话系统、叙事预测和问答等知识驱动的下游任务至关重要。

现有的 RE 模型通常依赖于带有昂贵注释的训练数据，这限制了它们的应用。为了应对这一问题，本文提出了一种新的方法——**SURE（Summarization as Relation Extraction）**，将 RE 转化为摘要任务，通过间接监督来提高 RE 的精度和资源效率。

![](/img/PaperNotes/Summarization_as_Indirect_Supervision_for_Relation_Extraction/img1.png)

图1展示了 SURE 的结构。具体来说，SURE 通过关系和句子转换技术将 RE 转化为摘要任务，并应用约束推理进行关系预测。我们采用实体信息口语化技术，突出包含实体信息的句子上下文，并将关系口语化为模板式的简短摘要。这样，转换后的RE输入和输出自然适合摘要模型。然后，我们通过在转换后的RE数据上进行微调，将摘要模型适配于RE任务。在推理过程中，设计了一种 Trie 评分技术来推断关系。通过这种方式，SURE 充分利用了摘要的间接监督，即使在资源匮乏的情况下也能获得精确的RE模型。

这项工作的贡献有两个方面。首先，据我们所知，这是首次研究利用摘要的间接监督进行RE。由于摘要的目标与 RE 自然对齐，它允许在不完全依赖直接任务注释的情况下训练出精确的 RE 模型，并在资源匮乏的情况下表现出色。其次，我们研究了有效桥接摘要和 RE 任务形式的输入转换技术，以及进一步增强基于摘要的RE推理的约束技术。我们的贡献通过在三个广泛使用的句子级 RE 数据集 TACRED、TACREV 和 SemEval 以及 TACRED 的三个低资源设置上的实验得到验证。我们观察到，SURE 在低资源设置下（使用10%的 TACRED 训练数据）优于各种基线。SURE 还在 TACRED 和 TACREV上 分别以75.1%和83.5%的 micro-F1 得分达到了SOTA 性能。我们还进行了全面的消融研究，展示了摘要的间接监督的有效性以及 SURE 输入转换的最佳选项。

> 文本摘要（Text Summarization）任务：旨在从长文本中提取出关键信息，生成简短的摘要。
>
> 有些类似于文本生成任务（Text Generation），常见模型多为 Decoder-Only 和 Encoder-Decoder，即以生成序列为任务目标。



## Method

### Relation and Sentence Conversion

**输入序列构建**：关系抽取关注于分析两个特定实体之间的互动，因此我们需要进一步处理源句子，以便摘要模型能够捕捉到更多的信息。SURE 探索了一系列句子处理技术，这些技术突出并整合了实体信息，旨在找到适合摘要任务的技术。实体信息包括实体名称、类型和范围，这对于推断关系非常有用。我们探索了两种处理源句子的策略。

- 实体标记：

  ![](/img/PaperNotes/Summarization_as_Indirect_Supervision_for_Relation_Extraction/img3.png)

- 实体信息表达：

  直接将实体信息描述为语言环境的增强部分。尽管这种技术不能编码实体范围信息，但它使输入数据更接近自然语言，而不是添加特殊标记。这与摘要的间接监督很好地对齐。

  ![](/img/PaperNotes/Summarization_as_Indirect_Supervision_for_Relation_Extraction/img4.png)

**关系表达**：摘要的目标通过一组简单的语义模板进行表达，如图 1 的关系表达子图所示。每个模板包含 {subj} 和 {obj} 占位符，用于填充句子中的主语和宾语实体提及。

### Inference

SURE 的推理过程首先应用 Trie 评分对每个关系的可能性进行排序，并设置实体类型约束。进一步校准评分，以在已知关系和 NA 关系之间进行选择性预测。

**Trie 评分**：摘要模型使用束搜索技术生成顺序输出，而关系抽取旨在找出输入中描述的关系。为了支持使用摘要模型进行关系预测，我们开发了一种推理方法，通过使用摘要模型作为评分代理来对每个关系候选进行排序。受 Trie 约束解码的启发，我们开发了一种 Trie 评分技术，允许对候选关系的语言化进行高效排序。我们的方法不是计算整个关系模板的概率进行排序，而是在 Trie 上进行遍历，并将每个关系候选的概率估计为 Trie 上的路径概率。

**类型约束推理**：类型约束推理在许多最近的工作中出现。通过从训练集中构建类型-关系映射，模型只能在给定实体类型的情况下预测有效关系，这显著缩小了候选关系集的大小。类型约束推理可以很容易地结合到 SURE 中，通过从 Trie 评分中修剪无效实体类型的模板。

**NA 关系的校准**：考虑到关系本体不可能穷尽任何实体之间的所有可能关系，训练好的 RE 模型的推理自然会遇到许多实例，其中没有表达 RP 的正关系。因此，强制模型在正关系和预测放弃之间进行选择性决策尤为重要。这通过 SURE 中的校准技术实现，其中为 NA 设置了一个评分阈值 $s$ 。



## Experiments

![](/img/PaperNotes/Summarization_as_Indirect_Supervision_for_Relation_Extraction/img2.png)

![](/img/PaperNotes/Summarization_as_Indirect_Supervision_for_Relation_Extraction/img5.png)
