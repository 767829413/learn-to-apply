# 最小路径和

## 题目

给定一个包含非负整数的 `m x n` 网格 `grid` ，请找出一条从左上角到右下角的路径，使得路径上的数字总和为最小。

说明：每次只能向下或者向右移动一步。

示例:

![spiraln.jpg](https://s2.loli.net/2022/07/11/K8vi2zh7Yy9TFx3.jpg)

```text
输入：grid = [[1,3,1],[1,5,1],[4,2,1]]
输出：7
解释：因为路径 1→3→1→1→1 的总和最小。
```

---

## code

```go
// Unique paths ii
func minPathSum(grid [][]int) int {
	n := len(grid)
	m := len(grid[0])
	if n == 0 || m == 0 {
		return 0
	}
	if m == n && n == 1 {
		return grid[0][0]
	}
	min := func(a, b int) int {
		if a > b {
			return b
		}
		return a
	}
	// 动态规划求解,先推到子问题到最终大问题的求解递推式
	// dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])
	dp := make([][]int, n)
	for k := range dp {
		dp[k] = make([]int, m)
	}
	// 最初解确定
	dp[0][0] = grid[0][0]
	// 向下走的初始最小解
	for i := 1; i < n; i++ {
		dp[i][0] = grid[i][0] + dp[i-1][0]
	}
	// 向右走的初始最小解
	for j := 1; j < m; j++ {
		dp[0][j] = grid[0][j] + dp[0][j-1]
	}

	for i := 1; i < n; i++ {
		for j := 1; j < m; j++ {
			// 按递推公式进行递推求解
			dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])
		}
	}
	return dp[n-1][m-1]
}
```
