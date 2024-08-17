---
title: "BPE算法"
date: 2024-08-17
aliases: ["/NLP"]
tags: ["NLP"]
categories: ["NLP"]
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

## 基本概念

Byte-Pair Encoding (BPE) 是一种常用于自然语言处理（NLP）的分词算法。BPE最初是一种数据压缩算法，由Philip Gage在1994年提出。在NLP中，BPE被用来将文本分割成子词（subword）单元，这样可以在处理未见过的单词时更有效。



## 工作原理

BPE的核心思想是通过多次迭代，将最常见的字符对（或子词对）合并成一个新的符号，直到词汇表达到预定的大小。具体步骤如下：

1. **初始化词汇表**：将所有单个字符作为初始词汇表。
2. **统计频率**：统计所有相邻字符对的出现频率。
3. **合并字符对**：找到出现频率最高的字符对，并将其合并成一个新的符号。
4. **更新词汇表**：将新的符号加入词汇表，并更新文本中的所有相应字符对。
5. **重复步骤2-4**：直到词汇表达到预定大小。



## 例子

假设我们有一个简单的文本：“banana banana”. 初始词汇表为：{b, a, n, }. 具体步骤如下：

1. **初始化**：词汇表为 {b, a, n, }。
2. **统计频率**：统计相邻字符对的频率，如 “ba” 出现2次，“an” 出现2次，“na” 出现2次。
3. **合并字符对**：选择频率最高的字符对 “an”，将其合并成一个新符号 “an”。更新后的文本为 “b an an a b an an a”。
4. **更新词汇表**：词汇表更新为 {b, a, n, an}。
5. **重复步骤2-4**：继续统计频率并合并，直到词汇表达到预定大小。



## Byte-Level BPE

在处理多语言文本时，BPE的一个问题是字符集可能会非常大。为了解决这个问题，可以使用Byte-Level BPE。Byte-Level BPE将每个字节视为一个字符，这样基础字符集的大小固定为256（即所有可能的字节值）。



## 优缺点

**优点**：

- **处理未见过的单词**：通过将单词分割成子词，可以更好地处理未见过的单词
- **词汇表大小可控**：可以通过设置预定大小来控制词汇表的大小

**缺点**：

- **语义信息丢失**：在某些情况下，分割后的子词可能会丢失部分语义信息
- **计算复杂度**：多次迭代合并字符对的过程可能会增加计算复杂度
