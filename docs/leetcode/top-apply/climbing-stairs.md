# 爬楼梯

## 题目

假设你正在爬楼梯。需要 n 阶你才能到达楼顶。

每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

示例:

```text
输入：n = 2
输出：2
解释：有两种方法可以爬到楼顶。
1. 1 阶 + 1 阶
2. 2 阶
```

---

## code

```go
// Climbing stairs
// 很简单的动态规划公式 dp[i] = dp[i-1] + dp[i-2]
func climbStairs(n int) int {
	if n < 3 {
		s :=[3]int{}
		s[0], s[1], s[2] = 0, 1, 2
		return s[n]
	}
	dp := make([]int, n+1)
	dp[0], dp[1], dp[2] = 0, 1, 2
	for i := 3; i <= n; i++ {
		dp[i] = dp[i-1]+dp[i-2]
	}
	return dp[n]
}
```
