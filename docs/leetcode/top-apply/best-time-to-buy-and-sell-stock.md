# 买卖股票的最佳时机

## 题目

给定一个数组 prices ，它的第 i 个元素 prices[i] 表示一支给定股票第 i 天的价格。

你只能选择 某一天 买入这只股票，并选择在 未来的某一个不同的日子 卖出该股票。设计一个算法来计算你所能获取的最大利润。

返回你可以从这笔交易中获取的最大利润。如果你不能获取任何利润，返回 0 。

示例:

```text
输入：[7,1,5,3,6,4]
输出：5
解释：在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
     注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格；同时，你不能在买入前卖出股票。

输入：prices = [7,6,4,3,1]
输出：0
解释：在这种情况下, 没有交易完成, 所以最大利润为 0。
```

---

## code

```go
// Best time to buy and sell stock
func maxProfit(prices []int) int {
	l, maxP, k, max := len(prices), 0, 1, func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	if l <= 1 {
		return 0
	}
	dp := make([][]int, k+1)
	for index := range dp {
		dp[index] = make([]int, l)
	}

	// 答案来源: https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/discuss/39608/a-clean-dp-solution-which-generalizes-to-k-transactions
	// j 表示 prices[j]的价格
	// f[k, j] 代表在价格[j]之前的最大利润（注意：不是以价格[j]结束），最多使用k个交易
	// f[k, j] = max(f[k, j-1], prices[j] - prices[jj] + f[k-1, jj]) {0 <= jj <= j-1}
	//         = max(f[k, j-1], prices[j] + max(f[k-1, jj] - prices[jj]))
	// f[0, j] = 0; 0次交易就是0利润
	// f[k, 0] = 0; 如果价格保持不变,不管交易多少次收益都是0
	for i := 1; i <= k; i++ {
		tmaxp := dp[i-1][0] - prices[0]
		for j := 1; j < l; j++ {
			dp[i][j] = max(dp[i][j-1], prices[j]+tmaxp)
			tmaxp = max(tmaxp, dp[i-1][j]-prices[j])
			maxP = max(maxP, dp[i][j])
		}
	}
	return maxP
}

func maxProfit2(prices []int) int {
	l := len(prices)
	if l < 2 {
		return 0
	}
	// 构建结果存储
	// dp[i][0] 表示第i天不持有股票
	// dp[i][1] 表示第i天持有股票
	// 状态转移方程
	// dp[i][0] = max(dp[i-1][0],dp[i-1][1]+prices[i])
	// dp[i][1] = max(dp[i-1][1],-prices[i])
	dp, max := make([][]int, l), func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	for k := range dp {
		dp[k] = make([]int, 2)
	}
	// 构建初解
	dp[0][0] = 0
	dp[0][1] = -prices[0]

	//遍历价格数组
	for i := 1; i < l; i++ {
		// 第i天不持有的最大收益应该是第i-1天不持有的最大收益和第i-1天持有股票,第i天卖出的最大收益之中的最大值
		dp[i][0] = max(dp[i-1][0], dp[i-1][1]+prices[i])
		// 第i天持有的最大收益应该是第i-1天持有的最大收益和第i-1天没有持有股票,第i天买入股票的最大收益之中的最大值
		dp[i][1] = max(dp[i-1][1], -prices[i])
	}
	// 结果肯定是卖出的收益
	return dp[l-1][0]
}
```
