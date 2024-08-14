---
title: "😺 Is All You Need——Transformer补充"
date: 2024-08-14
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

## 关于本文动机

Transformer主要内容请见 [Lecture 9: Transformer | KurongBlog (kurongtohsaka.github.io)](https://kurongtohsaka.github.io/posts/cs224n/lesson_9/)，对 Transformer 已经进行比较详细的介绍和讲解了，但还是有一些细节问题不好在该篇文章提及，所以单开一篇讨论。



## Q，K，V 的理解

假设我们想让所有的词都与第一个词 $v_1$ 相似，我们可以让 $v_1$ 作为查询。 然后，将该查询与句子中所有词进行点积，这里的词就是键。 所以查询和键的组合给了我们权重，接着再将这些权重与作为值的所有单词相乘。

通过下面的公式可以理解这个过程，并理解查询、键、值分别代表什么意思：
$$
softmax(QK)=W \\
WV=Y
$$
一种比较感性的理解：想要得到某个 $V$ 对应的某个可能的相似信息需要先 $Q$ 这个 $V$ 的 $K$ ，$QK$ 得到注意力分数，之后经过 softmax 平滑后得到概率 $W $，然后 $WV$ 后得到最终的相似信息 $Y$ 。



## Attention 机制

在数据库中，如果我们想通过查询 $q$ 和键 $k_i$ 检索某个值 $v_i$ 。注意力与这种数据库取值技术类似，但是以概率的方式进行的。

$$
attention(q,k,v)=\sum_isimilarity(q,k_i)v_i
$$

- 注意力机制测量查询 $q$ 和每个键值 $k_i$ 之间的相似性。
- 返回每个键值的权重代表这种相似性。
- 最后，返回所有值的加权组合作为输出。



## Mask 掩码

 在机器翻译或文本生成任务中，我们经常需要预测下一个单词出现的概率，这类任务我们一次只能看到一个单词。此时注意力只能放在下一个词上，不能放在第二个词或后面的词上。简而言之，注意力不能有非平凡的超对角线分量。

我们可以通过添加掩码矩阵来修正注意力，以消除神经网络对未来的了解。



## Multi-head Attention 多头注意力机制

“小美长得很漂亮而且人还很好” 。这里“人”这个词，在语法上与“小美”和“好”这些词存在某种意义或关联。这句话中“人”这个词需要理解为“人品”，说的是小美的人品很好。仅仅使用一个注意力机制可能无法正确识别这三个词之间的关联，这种情况下，使用多个注意力可以更好地表示与“人”相关的词。这减少了注意力寻找所有重要词的负担，增加找到更多相关词的机会。



## 位置编码

在任何句子中，单词一个接一个地出现都蕴含着重要意义。如果句子中的单词乱七八糟，那么这句话很可能没有意义。但是当 Transformer 加载句子时，它不会按顺序加载，而是并行加载。由于 Transformer 架构在并行加载时不包括单词的顺序，因此我们必须明确定义单词在句子中的位置。这有助于 Transformer 理解句子词与词之间的位置。这就是位置嵌入派上用场的地方。位置嵌入是一种定义单词位置的向量编码。在进入注意力网络之前，将此位置嵌入添加到输入嵌入中。

作者使用交替正余弦函数来定义位置嵌入：

![](/img/NLP/img16.png)



## 代码实现

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.utils.data as data
import math
import copy


# 多头注意力
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads):
        super(MultiHeadAttention, self).__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        
        self.W_q = nn.Linear(d_model, d_model)
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)
        
    def scaled_dot_product_attention(self, Q, K, V, mask=None):
        attn_scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        if mask is not None:
            attn_scores = attn_scores.masked_fill(mask == 0, -1e9)
        attn_probs = torch.softmax(attn_scores, dim=-1)
        output = torch.matmul(attn_probs, V)
        return output
        
    def split_heads(self, x):
        batch_size, seq_length, d_model = x.size()
        return x.view(batch_size, seq_length, self.num_heads, self.d_k).transpose(1, 2)
        
    def combine_heads(self, x):
        batch_size, _, seq_length, d_k = x.size()
        return x.transpose(1, 2).contiguous().view(batch_size, seq_length, self.d_model)
        
    def forward(self, Q, K, V, mask=None):
        Q = self.split_heads(self.W_q(Q))
        K = self.split_heads(self.W_k(K))
        V = self.split_heads(self.W_v(V))
        
        attn_output = self.scaled_dot_product_attention(Q, K, V, mask)
        output = self.W_o(self.combine_heads(attn_output))
        return output


# 位置前馈网络      
class PositionWiseFeedForward(nn.Module):
    def __init__(self, d_model, d_ff):
        super(PositionWiseFeedForward, self).__init__()
        self.fc1 = nn.Linear(d_model, d_ff)
        self.fc2 = nn.Linear(d_ff, d_model)
        self.relu = nn.ReLU()

    def forward(self, x):
        return self.fc2(self.relu(self.fc1(x)))

      
# 位置编码      
class PositionalEncoding(nn.Module):
    def __init__(self, d_model, max_seq_length):
        super(PositionalEncoding, self).__init__()
        
        pe = torch.zeros(max_seq_length, d_model)
        position = torch.arange(0, max_seq_length, dtype=torch.float).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * -(math.log(10000.0) / d_model))
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        self.register_buffer('pe', pe.unsqueeze(0))
        
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]


# 编码器      
class EncoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout):
        super(EncoderLayer, self).__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.feed_forward = PositionWiseFeedForward(d_model, d_ff)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask):
        attn_output = self.self_attn(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        ff_output = self.feed_forward(x)
        x = self.norm2(x + self.dropout(ff_output))
        return x


# 解码器      
class DecoderLayer(nn.Module):
    def __init__(self, d_model, num_heads, d_ff, dropout):
        super(DecoderLayer, self).__init__()
        self.self_attn = MultiHeadAttention(d_model, num_heads)
        self.cross_attn = MultiHeadAttention(d_model, num_heads)
        self.feed_forward = PositionWiseFeedForward(d_model, d_ff)
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
        self.norm3 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, enc_output, src_mask, tgt_mask):
        attn_output = self.self_attn(x, x, x, tgt_mask)
        x = self.norm1(x + self.dropout(attn_output))
        attn_output = self.cross_attn(x, enc_output, enc_output, src_mask)
        x = self.norm2(x + self.dropout(attn_output))
        ff_output = self.feed_forward(x)
        x = self.norm3(x + self.dropout(ff_output))
        return x

      
class Transformer(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, d_model, num_heads, num_layers, d_ff, max_seq_length, dropout):
        super(Transformer, self).__init__()
        self.encoder_embedding = nn.Embedding(src_vocab_size, d_model)
        self.decoder_embedding = nn.Embedding(tgt_vocab_size, d_model)
        self.positional_encoding = PositionalEncoding(d_model, max_seq_length)

        self.encoder_layers = nn.ModuleList([EncoderLayer(d_model, num_heads, d_ff, dropout) for _ in range(num_layers)])
        self.decoder_layers = nn.ModuleList([DecoderLayer(d_model, num_heads, d_ff, dropout) for _ in range(num_layers)])

        self.fc = nn.Linear(d_model, tgt_vocab_size)
        self.dropout = nn.Dropout(dropout)

    def generate_mask(self, src, tgt):
        src_mask = (src != 0).unsqueeze(1).unsqueeze(2)
        tgt_mask = (tgt != 0).unsqueeze(1).unsqueeze(3)
        seq_length = tgt.size(1)
        nopeak_mask = (1 - torch.triu(torch.ones(1, seq_length, seq_length), diagonal=1)).bool()
        tgt_mask = tgt_mask & nopeak_mask
        return src_mask, tgt_mask

    def forward(self, src, tgt):
        src_mask, tgt_mask = self.generate_mask(src, tgt)
        src_embedded = self.dropout(self.positional_encoding(self.encoder_embedding(src)))
        tgt_embedded = self.dropout(self.positional_encoding(self.decoder_embedding(tgt)))

        enc_output = src_embedded
        for enc_layer in self.encoder_layers:
            enc_output = enc_layer(enc_output, src_mask)

        dec_output = tgt_embedded
        for dec_layer in self.decoder_layers:
            dec_output = dec_layer(dec_output, enc_output, src_mask, tgt_mask)

        output = self.fc(dec_output)
        return output

      
if __name__ == '__main__':
  src_vocab_size = 5000
  tgt_vocab_size = 5000
  d_model = 512
  num_heads = 8
  num_layers = 6
  d_ff = 2048
  max_seq_length = 100
  dropout = 0.1

  transformer = Transformer(src_vocab_size, tgt_vocab_size, d_model, num_heads, num_layers, d_ff, max_seq_length, dropout)

  # 生成随机样本数据
  src_data = torch.randint(1, src_vocab_size, (64, max_seq_length))  
  tgt_data = torch.randint(1, tgt_vocab_size, (64, max_seq_length))
  
  criterion = nn.CrossEntropyLoss(ignore_index=0)
  optimizer = optim.Adam(transformer.parameters(), lr=0.0001, betas=(0.9, 0.98), eps=1e-9)

  transformer.train()

  for epoch in range(100):
      optimizer.zero_grad()
      output = transformer(src_data, tgt_data[:, :-1])
      loss = criterion(output.contiguous().view(-1, tgt_vocab_size), tgt_data[:, 1:].contiguous().view(-1))
      loss.backward()
      optimizer.step()
      print(f"Epoch: {epoch+1}, Loss: {loss.item()}")

```

