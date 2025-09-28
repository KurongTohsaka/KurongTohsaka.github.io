---
title: "几道反复做了好几遍的一维DP"
date: 2025-09-28
aliases: ["/Algorithm Learning"]
tags: ["DP"]
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
searchHidden: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## [70.爬楼梯](https://leetcode.cn/problems/climbing-stairs/description/?envType=study-plan-v2&envId=top-interview-150)

```go
func climbStairs(n int) int {
	cache := make([]int, n+1)
	cache[0], cache[1] = 1, 1

	for i := 2; i <= n; i++ {
		cache[i] = cache[i-1] + cache[i-2]
	}

	return cache[n]
}
```

最简单的一维 DP 题，遇见属于白给，最标准的没有套路的 DP，完全不需要解释。



## [198.打家劫舍](https://leetcode.cn/problems/house-robber/?envType=study-plan-v2&envId=top-interview-150)

```go
func rob(nums []int) int {
	cache := make([]int, len(nums)+1)
	cache[0], cache[1] = 0, nums[0]

	for i := 2; i <= len(nums); i++ {
		cache[i] = max(nums[i-1]+cache[i-2], cache[i-1])
	}

	return cache[len(nums)]
}
```

经典打家劫舍，难度同样属于白给。



##  [139. 单词拆分](https://leetcode.cn/problems/word-break/) 

```go
func wordBreak(s string, wordDict []string) bool {
	words := make(map[string]bool, len(wordDict))
	for _, w := range wordDict {
		words[w] = true
	}

	n := len(s)
	cache := make([]bool, n+1)
	cache[0] = true

	for i := 1; i <= n; i++ {
		for j := 0; j < i; j++ {
			if cache[j] && words[s[j:i]] {
				cache[i] = true
				break
			}
		}
	}

	return cache[n]
}
```

零钱兑换变体，完全背包问题。有一定难度，属于做了多少遍都会忘的题。

别忘记一定会有一个双循环，内层循环用来遍历要求的东西。该题内层循环是遍历 `s[j:i]` 子串，同时与 `words` 词表进行比对。当 `cache[j]` 存在且 `words[s[j:i]]` 存在，也就意味着这两样东西可以拼接在一起，那此时 `cache[i]` 就可以存在了。

 

 ## [322. 零钱兑换](https://leetcode.cn/problems/coin-change/)

```go
func coinChange(coins []int, amount int) int {
	cache := make([]int, amount+1)

	for i := 1; i <= amount; i++ {
		cache[i] = math.MaxInt
		for j := 0; j < len(coins); j++ {
			if i >= coins[j] && cache[i-coins[j]] != math.MaxInt {
				cache[i] = min(cache[i], cache[i-coins[j]]+1)
			}
		}
	}

	if cache[amount] == math.MaxInt {
		return -1
	}

	return cache[amount]
}
```

标准的完全背包问题，有一定难度。

这里的内层循环是遍历 `coins`，同时做一个条件判断：当前外层的金额 `i` 是否不小于硬币面值 && 金额 `i - coin` 可以被 `coins` 表示。如果通过判断，则通过状态转移方程 `cache[i] = min(cache[i], cache[i-coin]+1)` 来进行计算。



##   [300. 最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/) 

DP 解法：

```go
func lengthOfLIS(nums []int) int {
	n := len(nums)
	cache := make([]int, n)
	for i := range n {
		cache[i] = 1
	}

	for i := 1; i < n; i++ {
		for j := 0; j < i; j++ {
			if nums[i] > nums[j] {
				cache[i] = max(cache[i], cache[j]+1)
			}
		}
	}

	return slices.Max(cache)
}
```

DP 解法类似上面的 LC139 。

要计算 `cache[i]` ，需要看 `i` 之前的位置 `j` ，如果 `nums[i] > nums[j]` ，意味着 `nums[i]` 可以接在以 `nums[j]` 结尾的那个递增子序列的后面，再根据状态转移方程求解 `cache[i] = max(cache[i], cache[j]+1)` 。

更优的二分查找解法：

```go
func lengthOfLIS(nums []int) int {
	array := make([]int, 0)

	for _, num := range nums {
		i := binarySearch(array, num)
		if i == len(array) {
			array = append(array, num)
		} else {
			array[i] = num
		}
	}

	return len(array)
}

func binarySearch(nums []int, target int) int {
	left, right := 0, len(nums)-1
	for left <= right {
		mid := left + (right-left)/2
		if nums[mid] < target {
			left = mid + 1
		} else if nums[mid] > target {
			right = mid - 1
		} else {
			return mid
		}
	}
	return left
}
```

二分查找解法比较好理解，主要是在主函数中的处理：

- 当 `i == len(array)` 时，表示有一个新的元素要进入候选序列中，否则只更改 `array[i] = num` 。
