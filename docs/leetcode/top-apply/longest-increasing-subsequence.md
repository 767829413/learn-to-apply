# 最长递增子序列

## 题目

给你一个整数数组 nums ，找到其中最长严格递增子序列的长度。

子序列 是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。例如，[3,6,2,7] 是数组 [0,3,1,6,2,2,7] 的子序列。

示例:

```text
输入：nums = [10,9,2,5,3,7,101,18]
输出：4
解释：最长递增子序列是 [2,3,7,101]，因此长度为 4 。
```

---

## code

```go
// Longest increasing subsequence
func lengthOfLIS(nums []int) int {
	l := len(nums)
	if l < 2 {
		return l
	}
	// 构建初始dp数组,基础值为1
	dp := make([]int, l)
	for k := range dp {
		dp[k] = 1
	}
	max := func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	res := math.MinInt
	// 位置i的dp[i]最大值是 j表示[0,i)的值
	// 1. nums[i] > nums[j] 则i是所在位置j的严格后序,此时对应dp[j]++
	// 2. nums[i] <= nums[j] 不满足条件则跳出
	// dp[i] = max(dp[i],dp[j] + 1) j => [0,j)
	for i := 0; i < l; i++ {
		for j := 0; j < i; j++ {
			if nums[i] > nums[j] {
				dp[i] = max(dp[i], dp[j]+1)
			}
		}
		res = max(dp[i], res)
	}
	return res
}
```
