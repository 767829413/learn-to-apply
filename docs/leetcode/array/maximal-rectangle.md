# 最大矩形

## 题目

给定一个仅包含 `0` 和 `1` 、大小为 `rows x cols` 的二维二进制矩阵，找出只包含 `1` 的最大矩形，并返回其面积。

示例:

![histogram.jpg](https://s2.loli.net/2022/07/27/kaIleOcE2Mv8RDZ.jpg)

```text
输入：matrix = [["1","0","1","0","0"],["1","0","1","1","1"],["1","1","1","1","1"],["1","0","0","1","0"]]
输出：6
解释：最大矩形如上图所示。

输入：matrix = []
输出：0

输入：matrix = [["0"]]
输出：0

输入：matrix = [["1"]]
输出：1

输入：matrix = [["0","0"]]
输出：0
```

---

## code

```go
// Maximal rectangle
func maximalRectangle(matrix [][]byte) int {
	n := len(matrix)
	m := len(matrix[0])

	maxArea := math.MinInt
	max := func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	// 通过寻找最大矩形的题解来解决当前问题
	largestRectangleArea := func(heights []int) int {
		l := len(heights)
		if l == 0 {
			return 0
		}
		if l == 1 {
			return heights[0]
		}
		// 搜索数组,以当前高度,向左右寻找边界
		// 边界必定是小于等于当前高度所对应的高度对应的index
		// 左右边界的差值(边界肯定不包含在内)需要减一才是宽度

		left, right, stack := make([]int, l), make([]int, l), []int{}
		// 寻找左边界
		for i := 0; i < l; i++ {
			for len(stack) > 0 && heights[stack[len(stack)-1]] >= heights[i] {
				stack = stack[:len(stack)-1]
			}
			if len(stack) == 0 {
				left[i] = -1
			} else {
				left[i] = stack[len(stack)-1]
			}
			// 当前位置入栈
			stack = append(stack, i)
		}
		stack = []int{}
		for i := l - 1; i >= 0; i-- {
			for len(stack) > 0 && heights[stack[len(stack)-1]] >= heights[i] {
				stack = stack[:len(stack)-1]
			}
			if len(stack) == 0 {
				right[i] = l
			} else {
				right[i] = stack[len(stack)-1]
			}
			stack = append(stack, i)
		}
		ans := 0
		max := func(a, b int) int {
			if a > b {
				return a
			}
			return b
		}
		for i := 0; i < l; i++ {
			ans = max(ans, ((right[i] - left[i] - 1) * heights[i]))
		}
		return ans
	}

	for row := 0; row < n; row++ {
		input := make([]int, m)
		for col := 0; col < m; col++ {
			// 构建求解入参
			if row == 0 {
				if matrix[row][col] == '1' {
					input[col] = 1
				} else {
					input[col] = 0
				}
			} else {
				if matrix[row][col] == '0' {
					input[col] = 0
				} else {
					for now := row; now >= 0; now-- {
						if matrix[now][col] == '0' {
							break
						}
						input[col]++
					}
				}
			}
		}
		maxArea = max(maxArea, largestRectangleArea(input))
	}
	return maxArea
}

// 暴力求解
// Maximal rectangle
func maximalRectangle(matrix [][]byte) int {
	n := len(matrix)
	m := len(matrix[0])

	min := func(a, b int) int {
		if a > b {
			return b
		}
		return a
	}
	max := func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	with := make([][]int, n)
	maxArea := 0
	for k := range with {
		with[k] = make([]int, m)
	}
	// 暴力枚举
	for row := 0; row < n; row++ {
		for col := 0; col < m; col++ {
			// 统计所有位置为1的最小宽度
			// 所谓最小宽度是连续的中间不为0
			if matrix[row][col] == '0' {
				continue
			} else {
				if col == 0 {
					with[row][col] = 1
				} else {
					with[row][col] = with[row][col-1] + 1
				}
			}
			midWith := with[row][col]
			// 通过当前行,向上扩充(就是增加高度)
			// 计算面积.选出最大的
			for now := row; now >= 0; now-- {
				h := row - now + 1
				midWith = min(midWith, with[now][col])
				maxArea = max(maxArea, h*midWith)
			}
		}
	}
	return maxArea
}
```
