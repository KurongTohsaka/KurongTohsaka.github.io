---
title: "Leetcode-75 总结"
date: 2025-03-02
aliases: ["/Algorithm Learning"]
tags: ["Leetcode"]
categories: ["Algorithm Learning"]
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

# Leetcode-75 总结

**代码链接：[KurongTohsaka/my-lc75-go](https://github.com/KurongTohsaka/my-lc75-go)**

## Array&String

- 151 反转字符串中的单词：双指针。
- 238 除自身以外数组的乘积：线性 DP 。
- 334 递增的三元子序列：双指针。
- 345 反转字符串中的元音字母：双指针。
- 443 压缩字符串：双指针。
- 605 种花问题：列出三种种花情况，依次进行处理即可。
- 1071 字符串的最大公因式：根据特殊的 `str1 + str2 == str2 + str1` 公式和辗转相除法解题。
- 1431 拥有最多糖果的孩子：先找最大值，然后进行比较。
- 1768 交替合并字符串：取较小长度最为切割位置，然后交替合并，最后处理剩余部分。



## DoublePointers

- 11 接水问题：双指针的首尾指针。
- 283 移动零：双指针的快慢指针。
- 392 判断子序列：双指针的快慢指针。
- 1679 K 和数对的最大数目：解法与两数之和相似，使用哈希表记录为匹配数组次数，然后更新 `hashMap[k-v]` 的数值。



## SlidingWindow

- 643 子数组最大平均数-1 ：定长滑动窗口，应用通用解法，即新元素进入、更新统计量、`right-left+1` 元素离开三步。
- 1004 最大连续1的个数-2 ：变长滑动窗口，通过快慢指针实现。
- 1456 定长子串中元音的最大数目：定长滑动窗口，通用解法。
- 1493 删掉一个元素后全为1的最长子数组：变长滑动窗口，解法与 1004 高度相似。



## PrefixSum

- 724 寻找数组的中心下标：双指针可解，不过还是推导出的公式更快。
- 1732 找到最高海拔：很简单，直接秒。
