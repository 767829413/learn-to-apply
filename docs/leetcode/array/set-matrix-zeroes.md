# 矩阵置零

## 题目

给定一个 `m x n` 的矩阵，如果一个元素为 `0` ，则将其所在行和列的所有元素都设为 `0` 。请使用 `原地` 算法。

示例:

![spiraln.jpg](https://s2.loli.net/2022/07/14/mXBCc45ipEDYf6G.jpg)

```text
输入：matrix = [[1,1,1],[1,0,1],[1,1,1]]
输出：[[1,0,1],[0,0,0],[1,0,1]]
```

---

## code

```go
// Set matrix zeroes
func setZeroes(matrix [][]int) {
	n := len(matrix)
	m := len(matrix[0])

	if n == 0 || m == 0 {
		return
	}
	// 判断行列是否有0标志
	col0Flag, row0Flag := false, false
	// 判断行是否有0
	for i := 0; i < n; i++ {
		if matrix[i][0] == 0 {
			row0Flag = true
			break
		}
	}

	// 判断列是否为0
	for j := 0; j < m; j++ {
		if matrix[0][j] == 0 {
			col0Flag = true
			break
		}
	}

	// 然后利用第一行第一列作为记录来标记
	for i := 1; i < n; i++ {
		for j := 1; j < m; j++ {
			if matrix[i][j] == 0 {
				matrix[0][j], matrix[i][0] = 0, 0
			}
		}
	}

	// 根据第一行第一列的标记开始置0
	for i := 1; i < n; i++ {
		for j := 1; j < m; j++ {
			if matrix[0][j] == 0 || matrix[i][0] == 0 {
				matrix[i][j] = 0
			}
		}
	}

	// 最后根绝条件将第一行第一列置0
	if row0Flag {
		for i := 0; i < n; i++ {
			matrix[i][0] = 0
		}
	}

	if col0Flag {
		for j := 0; j < m; j++ {
			matrix[0][j] = 0
		}
	}
}
```
