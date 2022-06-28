# N 皇后

## 题目

按照国际象棋的规则，皇后可以攻击与之处在同一行或同一列或同一斜线上的棋子。

`n 皇后问题` 研究的是如何将 `n` 个皇后放置在 `n×n` 的棋盘上，并且使皇后彼此之间不能相互攻击。

给你一个整数 `n` ，返回所有不同的 `n 皇后问题` 的解决方案。

每一种解法包含一个不同的 `n 皇后问题` 的棋子放置方案，该方案中 `'Q'` 和 `'.'` 分别代表了皇后和空位。

示例:

![queens.jpg](https://s2.loli.net/2022/06/28/NdJ86FLErtDmSjC.jpg)

```text
输入：n = 4
输出：[[".Q..","...Q","Q...","..Q."],["..Q.","Q...","...Q",".Q.."]]
解释：如上图所示，4 皇后问题存在两个不同的解法。

输入：n = 1
输出：[["Q"]]
```

---

## code

```go
func solveNQueens(n int) [][]string {
	if n == 0 {
		return [][]string{{""}}
	}
	if n == 1 {
		return [][]string{{"Q"}}
	}
	box := [][]int{}
	// 每一列的使用
	col := make([]bool, n)
	// 主对角线的使用 row(行) col(列) 关系是 row - col => [-3~3],总体+3,[0~6]
	main := make([]bool, 2*n-1)
	// 副对角线的使用 row(行) col(列) 关系是 row + col => [0~6]
	vice := make([]bool, 2*n-1)

	var dfs func(row int, path []int)
	dfs = func(row int, path []int) {
		if row == n {
			newPath := make([]int, len(path))
			copy(newPath, path)
			box = append(box, newPath)
			return
		}
		for i := 0; i < n; i++ {
			if col[i] || main[row-i+n-1] || vice[row+i] {
				continue
			}
			path = append(path, i)
			// 记录使用状态
			col[i] = true
			main[row-i+n-1] = true
			vice[row+i] = true

			dfs(row+1, path)

			// 状态还原
			col[i] = false
			main[row-i+n-1] = false
			vice[row+i] = false
			path = path[:len(path)-1]
		}
	}
	dfs(0, []int{})
	ans := make([][]string, len(box))
	// 将获取的数组转化为字符串
	for k, v := range box {
		ans[k] = []string{}
		for _, index := range v {
			tmp := []byte{}
			for j := 0; j < n; j++ {
				if j == index {
					tmp = append(tmp, 'Q')
				} else {
					tmp = append(tmp, '.')
				}
			}
			ans[k] = append(ans[k], string(tmp))
		}
	}
	return ans
}
```
