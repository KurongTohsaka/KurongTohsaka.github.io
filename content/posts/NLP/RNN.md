---
title: "RNN速成（一）"
date: 2024-07-06
aliases: ["/NLP"]
tags: ["NLP", "RNN"]
categories: ["NLP"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "了解RNN本体"
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

循环神经网络 (RNN) 是一种使用序列数据或时序数据的人工神经网络。其最大特点是网络中存在着环，使得信息能在网络中进行循环，实现对序列信息的存储和处理。

循环神经网络 (RNN) 的另一个显著特征是它们在每个网络层中共享参数。 虽然前馈网络的每个节点都有不同的权重，但循环神经网络在每个网络层都共享相同的权重参数。



## 网络结构

RNN 不是刚性地记忆所有固定长度的序列，而是通过隐藏状态来存储之前时间步的信息。

![](/img/NLP/img4.png)

同时，RNN 还能按**时间序列展开循环**为如下形式：

![](/img/NLP/img5.png)

以上架构不仅揭示了 RNN 的实质：上一个时刻的网络状态将会作用于（影响）到下一个时刻的网络状态，还表明 RNN 和序列数据密切相关。同时，RNN 要求每一个时刻都有一个输入，但是不一定每个时刻都需要有输出。

![](/img/NLP/img6.png)

如上图所示，隐含层的计算公式如下：
$$
s_t=f \ (U_{x_t}+W_{s_{t-1}})
$$
其中， $f$ 为激活函数。



## 训练方法

RNN 利用随时间推移的反向传播 (BPTT) 算法来确定梯度，这与传统的反向传播略有不同，因为它特定于序列数据。 BPTT 的原理与传统的反向传播相同，模型通过计算输出层与输入层之间的误差来训练自身。 这些计算帮助我们适当地调整和拟合模型的参数。 BPTT 与传统方法的不同之处在于，BPTT 会在每个时间步长对误差求和。

通过这个过程，RNN 往往会产生两个问题，即梯度爆炸和梯度消失。 这些问题由梯度的大小定义，也就是损失函数沿着错误曲线的斜率。 如果梯度过小，它会更新权重参数，让梯度继续变小，直到变得可以忽略，即为 0。 发生这种情况时，算法就不再学习。 如果梯度过大，就会发生梯度爆炸，这会导致模型不稳定。 在这种情况下，模型权重会变得太大，并最终被表示为 NaN。 



## RNN的变体

这里只是一个简介，详情见 《RNN速成（二）》

### LSTM

这是一种比较流行的 RNN 架构，由 Sepp Hochreiter 和 Juergen Schmidhuber 提出，用于解决梯度消失问题。LSTM 在神经网络的隐藏层中包含一些“元胞”(cell)，共有三个门：一个输入门、一个输出门和一个遗忘门。 这些门控制着预测网络中的输出所需信息的流动。

### GRU

这种 RNN 变体类似于 LSTM，因为它也旨在解决 RNN 模型的短期记忆问题。 但它不使用“元胞状态”来调节信息，而是使用隐藏状态；它不使用三个门，而是两个：一个重置门和一个更新门。 类似于 LSTM 中的门，重置门和更新门控制要保留哪些信息以及保留多少信息。
