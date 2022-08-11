# 最大子数组和

## 题目

给你一个整数数组 nums ，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

子数组 是数组中的一个连续部分。

示例:

```text
输入：nums = [-2,1,-3,4,-1,2,1,-5,4]
输出：6
解释：连续子数组 [4,-1,2,1] 的和最大，为 6 。

输入：nums = [1]
输出：1
```

---

## code

```go
// Maximum subarray
func maxSubArray(nums []int) int {
	l := len(nums)
	if l == 1 {
		return nums[0]
	}
	// 构建动态规划数组
	dp := make([]int, l)
	// 定义初解和最大值临时变量
	dp[0] = nums[0]
	m, max := math.MinInt, func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	m = max(dp[0], m)
	for i := 1; i < l; i++ {
		// fmt.Println(dp)
		// 如果上一步是大于0,那就可以累加
		if dp[i-1] > 0 {
			dp[i] = nums[i] + dp[i-1]
		} else {
			dp[i] = nums[i]
		}
		m = max(dp[i], m)
	}
	return m
}
```
