# 螺旋矩阵

## 题目

给你一个 `m` 行 `n` 列的矩阵 `matrix` ，请按照 `顺时针螺旋顺序` ，返回矩阵中的所有元素。

示例:

![spiral1.jpg](https://s2.loli.net/2022/06/30/O5IAJKvQwCnucWT.jpg)

```text
输入：matrix = [[1,2,3],[4,5,6],[7,8,9]]
输出：[1,2,3,6,9,8,7,4,5]
```

![spiral.jpg](https://s2.loli.net/2022/06/30/Y6AI9JDe38WKf2F.jpg)

```text
输入：matrix = [[1,2,3,4],[5,6,7,8],[9,10,11,12]]
输出：[1,2,3,4,8,12,11,10,9,5,6,7]
```

---

## code

```go
func spiralOrder(matrix [][]int) []int {
	box := []int{}
	l := len(matrix)
	if l == 0 {
		return box
	}
	// 构建上下,左右边界
	rowU, rowrD, colL, colR := 0, l-1, 0, len(matrix[0])-1
	// 向右走,向下走,向左走,向上走
	for {
		// 循环退出条件
		if colL > colR {
			break
		}

		for j := colL; j <= colR; j++ {
			box = append(box, matrix[rowU][j])
		}
		rowU++

		// 循环退出条件
		if rowU > rowrD {
			break
		}

		for i := rowU; i <= rowrD; i++ {
			box = append(box, matrix[i][colR])
		}
		colR--

		// 循环退出条件
		if colL > colR {
			break
		}

		for j := colR; j >= colL; j-- {
			box = append(box, matrix[rowrD][j])
		}
		rowrD--

		// 循环退出条件
		if rowU > rowrD {
			break
		}

		for i := rowrD; i >= rowU; i-- {
			box = append(box, matrix[i][colL])
		}
		colL++
	}
	return box
}
```
