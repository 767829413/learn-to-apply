# 岛屿数量

## 题目

给你一个由 `'1'（陆地）`和 `'0'（水）`组成的的二维网格，请你计算网格中岛屿的数量。

岛屿总是被水包围，并且每座岛屿只能由水平方向和/或竖直方向上相邻的陆地连接形成。

此外，你可以假设该网格的四条边均被水包围。

示例:

```text
输入：grid = [
  ["1","1","1","1","0"],
  ["1","1","0","1","0"],
  ["1","1","0","0","0"],
  ["0","0","0","0","0"]
]
输出：1
```

---

## code

```go
// Number of islands
func numIslands(grid [][]byte) int {
	n, m, res := len(grid), len(grid[0]), 0
	// 定义深度优先函数
	var dfs func(grid [][]byte, i, j int) int
	dfs = func(grid [][]byte, i, j int) int {
		// 判断是不是越界或海水
		// 0 海水
		// 1 陆地
		// 2 已经遍历过的陆地
		if i >= n || j >= m || i < 0 || j < 0 || grid[i][j] != '1' {
			return 0
		}
		if grid[i][j] == '1' {
			// 标记已经遍历过了
			grid[i][j] = '2'
		}
		return 1 + dfs(grid, i+1, j) + dfs(grid, i-1, j) + dfs(grid, i, j+1) + dfs(grid, i, j-1)

	}
	for i := 0; i < n; i++ {
		for j := 0; j < m; j++ {
			num := dfs(grid, i, j)
			if num != 0 {
				res++
			}
		}
	}
	return res
}
```
