---
title: "Lecture 4: Dependency parsing"
date: 2024-07-04
aliases: ["/CS224N"]
tags: ["CS224N", "NLP"]
categories: ["CS224N"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "第四讲：依存句法分析"
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

## Two views of linguistic structure

### Context-free grammars (CFGs)

Phrase structure organizes words into nested constituents.

![](/img/CS224N/lesson_4/img1.png)

### Dependency structure

Dependency structure shows which words depend on (modify, attach to, or are  arguments of) which other words.

> modify：修饰词，attach to：连接词

### Why do we need sentence structure?

- Humans communicate complex ideas by composing words together  into bigger units to convey complex meanings.
- Listeners need to work out what modifies [attaches to] what
- A model needs to understand sentence structure in order to be  able to interpret language correctly



## Linguistic Ambiguities

- Prepositional phrase attachment ambiguity 介词短语附着歧义
- Coordination scope ambiguity 对等范围歧义
- Adjectival/Adverbial Modifier Ambiguity 形容词修饰语歧义
- Verb Phrase (VP) attachment ambiguity 动词短语依存歧义

Dependency paths identify semantic relations 依赖路径识别语义关系 help extract semantic interpretation.

![](/img/CS224N/lesson_4/img2.png)



## Dependency Grammar and Dependency Structure

![](/img/CS224N/lesson_4/img3.png)

Tesnière had them point from head to dependent – we follow that convention.

We usually add a fake ROOT so every word is a dependent of precisely 1 other node.

> 箭头指向依赖者 dependent。
>
> 通常添加⼀个伪根指向整个句子的头部，这样每个单词都精确地依赖于另⼀个节点。

### Dependency Conditioning Preferences

- Bilexical affinities 两个单词间的密切关系
- Dependency distance 依赖距离：主要是与相邻词
- Intervening material 介于中间的物质：如果中间的词语是动词或者标点，则两边的词语不太可能有依存
- Valency of heads：How many dependents on which side are usual for a head?



## Methods of Dependency Parsing

###  Arc-standard transition-based parser 标准弧转移算法

基于转换的算法有一个栈和一个词队列，除此之外还有一个已经 parse 的依存关系。

该算法包含以下操作：

- LEFT-ARC 栈顶和它下面的词构成依存关系，并且中心词是栈顶元素，把这两个词从栈中弹出，把这个依存关系加入到已parse的数据结构里，最后把中心词再加到栈中
- RIGHT-ARC 栈顶和它下面的词构成依存关系，中心词是下面的元素，把这两个词从栈中弹出，把这个依存关系加入到已parse的数据结构里，最后把中心词再加到栈中
- SHIFT 把队列中的一个词加入到栈顶

这个算法很简单，初始状态栈里只有一个root，而队列里是所有的词，然后循环直到状态是结束状态(栈中只有root，队列也为空)。

![](/img/CS224N/lesson_4/img4.png)

例子：

![](/img/CS224N/lesson_4/img6.png)

![](/img/CS224N/lesson_4/img5.png)

### Other Algorithms

- Greedy transition-based parsing
- MaltParser
- Conventional Feature Representation

### Evaluation of Dependency Parsing

- UAS (unlabeled attachment score) 指 无标记依存正确率 ，不考虑依存关系的类型，只考虑依存关系的对应单词是否正确，如父节点被正确识别的准确率；
- LAS (labeled attachment score) 指有标记依存正确率，同时考虑依存关系的对应单词以及依存关系类型，即词的父节点，以及与父节点的句法关系都被正确识别的概率。

### Neural dependency parsing

Just one word win. 
