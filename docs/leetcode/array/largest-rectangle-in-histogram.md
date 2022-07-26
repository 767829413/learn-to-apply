# 柱状图中最大的矩形

## 题目

给定 `n` 个非负整数，用来表示柱状图中各个柱子的高度。每个柱子彼此相邻，且宽度为 `1` 。

求在该柱状图中，能够勾勒出来的矩形的最大面积。

示例:

![histogram.jpg](https://s2.loli.net/2022/07/26/JZyAUGrEbu3zkhY.jpg)

```text
输入：heights = [2,1,5,6,2,3]
输出：10
解释：最大的矩形为图中红色区域，面积为 10
```

---

## code

```go
// Largest rectangle in histogram
func largestRectangleArea(heights []int) int {
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
```
