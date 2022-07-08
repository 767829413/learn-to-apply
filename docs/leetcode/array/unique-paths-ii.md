# 不同路径 II

## 题目

一个机器人位于一个 `m x n` 网格的左上角 `（起始点在下图中标记为 “Start” ）`。

机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角`（在下图中标记为 “Finish”）`。

现在考虑网格中有障碍物。那么从左上角到右下角将会有多少条不同的路径？

网格中的障碍物和空位置分别用 `1` 和 `0` 来表示。

示例:

![spiraln.jpg](https://s2.loli.net/2022/07/08/qfLQ1rGRbA2cNSv.jpg)

```text
输入：obstacleGrid = [[0,0,0],[0,1,0],[0,0,0]]
输出：2
解释：3x3 网格的正中间有一个障碍物。
从左上角到右下角一共有 2 条不同的路径：
1. 向右 -> 向右 -> 向下 -> 向下
2. 向下 -> 向下 -> 向右 -> 向右
```

---

## code

```go
// Unique paths ii
func uniquePathsWithObstacles(obstacleGrid [][]int) int {
	n := len(obstacleGrid)
	m := len(obstacleGrid[0])
	if n == 0 || m == 0 || obstacleGrid[0][0] == 1 {
		return 0
	}
	// 动态规划求解,先推到子问题到最终大问题的求解递推式
	// dp[i][j] = dp[i-1][j] + dp[i][j-1]
	dp := make([][]int, n)
	for k := range dp {
		dp[k] = make([]int, m)
	}
	// 最初解确定
	dp[0][0] = 1
	// 最左侧可达解,只要有障碍物那后续都不可达,可达只有一条路径
	for i := 1; i < n && obstacleGrid[i][0] != 1; i++ {
		dp[i][0] = 1
	}
	// 最上侧可达解,同上
	for j := 1; j < m && obstacleGrid[0][j] != 1; j++ {
		dp[0][j] = 1
	}

	for i := 1; i < n; i++ {
		for j := 1; j < m; j++ {
			// 当前位置是障碍物的,那必然不可达
			if obstacleGrid[i][j] == 1 {
				continue
			}
			// 按递推公式进行递推求解
			dp[i][j] = dp[i-1][j] + dp[i][j-1]
		}
	}
	return dp[n-1][m-1]
}
```
