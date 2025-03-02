---
title: "Continuous Bag of Words"
date: 2024-06-29
aliases: ["/NLP"]
tags: ["NLP", "word2vec"]
categories: ["NLP"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "CBOW的简单认知"
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

## CBOW

CBOW是 `continuous bag of words` 的缩写，中文译为“连续词袋模型”。它和 `skip-gram` 的区别在于其通过上下文去预测中心词。

模型结构如下。

![](/img/NLP/img3.png)

在 **Projection Layer** 中，会将上下文的所有词向量进行累加：
$$
X_w=\sum^{2c}_{i=1}v(Context(w)_i) \in \R^m
$$
在 **Output Layer** 中，使用到的函数是 **Hierarchical softmax 层级softmax** 。

**Hierarchical softmax** 是利用哈夫曼树结构来减少计算量的方式。它将 word 按词频作为结点权值来构建哈夫曼树。

哈夫曼树是一种二叉树，实际上就是在不断的做二分类。

具体公式推导见[Hierarchical Softmax（层次Softmax） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/56139075)。

