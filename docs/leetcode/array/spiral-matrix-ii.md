# 插入区间

## 题目

给你一个正整数 `n` ，生成一个包含 `1` 到 `n2` 所有元素，且元素按顺时针顺序螺旋排列的 `n x n` 正方形矩阵 `matrix` 。

示例:

![spiraln.jpg](https://s2.loli.net/2022/07/07/Fg47RhyNJ2cbfUG.jpg)

```text
输入：n = 3
输出：[[1,2,3],[8,9,4],[7,6,5]]

输入：n = 1
输出：[[1]]
```

---

## code

```go
// Spiral matrix ii
func generateMatrix(n int) [][]int {
	end := n * n
	if end == 1 {
		return [][]int{{1}}
	}
	box := make([][]int, n)
	for k := range box {
		box[k] = make([]int, n)
	}
	// 还是很简单的思路 向右走,向下走,向左走,向上走,控制好边界
	// 退出条件就是 左边界 > 右边界 上边界 > 下边界
	start, colL, colR, rowU, rowD := 1, 0, n-1, 0, n-1
	for {
		// 向右走
		for j := colL; j <= colR; j++ {
			box[rowU][j] = start
			start++
		}
		// 向右走完,上边界下移
		rowU++

		if rowU > rowD {
			break
		}
		// 向下走
		for i := rowU; i <= rowD; i++ {
			box[i][colR] = start
			start++
		}
		// 向下走完,右边界左移
		colR--

		if colL > colR {
			break
		}
		// 向左走
		for j := colR; j >= colL; j-- {
			box[rowD][j] = start
			start++
		}
		// 向左走完,下边界上移
		rowD--
		// 向上走
		for i := rowD; i >= rowU; i-- {
			box[i][colL] = start
			start++
		}
		// 向上走完,左边界右移
		colL++

		if colL > colR {
			break
		}
	}
	return box
}
```
