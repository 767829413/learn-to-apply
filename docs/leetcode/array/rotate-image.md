# 旋转图像

## 题目

给定一个 `n × n` 的二维矩阵 `matrix` 表示一个图像。请你将图像顺时针旋转 `90` 度。

你必须在 `原地` 旋转图像，这意味着你需要直接修改输入的二维矩阵。请不要 `使用另一个矩阵来旋转图像`。

示例:

![示例1](https://pic.imgdb.cn/item/62b69b2a094754312902e1f9.jpg)

```text
输入：matrix = [[1,2,3],[4,5,6],[7,8,9]]
输出：[[7,4,1],[8,5,2],[9,6,3]]
```

![示例2](https://pic.imgdb.cn/item/62b69ace0947543129026c4b.jpg)

```text
输入：matrix = [[5,1,9,11],[2,4,8,10],[13,3,6,7],[15,14,12,16]]
输出：[[15,13,2,5],[14,3,4,1],[12,6,8,9],[16,7,10,11]]
```

---

## code

```go
func rotate(matrix [][]int) {
	n := len(matrix)
	// 关键公式 90°旋转表示 matrix[row][col] <=> matrix[col][n-row-1]
	// 水平翻转 matrix[row][col] <=> matrix[n-row-1][col]
	// 主对角线翻转 matrix[n-row-1][col] <=> matrix[col][n-row-1]

	// 水平轴翻转 记住水平轴翻转是整体的一半
	for i := 0; i < (n >> 1); i++ {
		for j := 0; j < n; j++ {
			matrix[i][j], matrix[n-i-1][j] = matrix[n-i-1][j], matrix[i][j]
		}
	}

	// fmt.Println(matrix)
	// 对角线翻转 记住翻转的时候j<i
	for i := 0; i < n; i++ {
		for j := 0; j < i; j++ {
			matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
		}
	}
	// fmt.Println(matrix)
}
```
