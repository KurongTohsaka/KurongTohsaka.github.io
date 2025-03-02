---
title: "《基于MRC的设备故障命名实体识别方法》笔记"
date: 2024-08-06
aliases: ["/PaperNotes"]
tags: ["PaperNotes"]
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
    image: "cover/基于MRC的设备故障命名实体识别方法.png" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
searchHidden: false
---

## MRC

本论文关于设备故障领域的NER任务，将 NER 的序列标注问题重构为 MRC 问题。

MRC（Machine Reading Comprehension）机器阅读理解，与文生成和机器翻译任务类似。本论文使用该方法，根堆每个实体类型，设计了相应地自然语言形式的查询，通过上下文的查询来定位并提取实体。例如，针对故障代码 the machine suffers from error[#10482] 的命名实体识别，被形式化为 “文本中提到了哪种故障代码 ” 。

> 人话：先通过某个文生成/机器翻译模型将错误代码转化为中文，然后通过预定的错误代码进行映射

本文的主要工作如下： 

1. 将 NER 任务转化为 MRC 问题，采用自然语言查询来实现命名实体的识别。
2. 通过在查询中集成预设的命名实体类别信息，克服传统方法中实体类别语义信息缺少的限制，提升命名实体识别的准确性。 
3. 结合设备故障数据集的构建，开展相关的实 验对比，并进行定量分析，证明了本文方法的有效性。 



## 模型结构

![](/img/PaperNotes/基于MRC的设备故障命名实体识别方法_徐鹏/img1.png)

- 编码层为 ALBERT 
- 特征提取为 BiLSTM 
- 分类器为两个sigmoid，分别预测实体的起始索引、结束索引



## 数据集

![](/img/PaperNotes/基于MRC的设备故障命名实体识别方法_徐鹏/img2.png)



## 对比试验

模型为以下几类：

![](/img/PaperNotes/基于MRC的设备故障命名实体识别方法_徐鹏/img4.png)

结果：

![](/img/PaperNotes/基于MRC的设备故障命名实体识别方法_徐鹏/img3.png)



## 消融实验

![](/img/PaperNotes/基于MRC的设备故障命名实体识别方法_徐鹏/img5.png)
