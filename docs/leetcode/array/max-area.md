# 盛最多水的容器

## 题目

给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。

找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

说明：你不能倾斜容器。

示例:

![question_11.jpg](https://s2.loli.net/2022/06/09/2PZR3eLTQc46oKV.jpg)

```text
输入：[1,8,6,2,5,4,8,3,7]
输出：49 
解释：图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。
```

---

## code

```go
func maxArea(height []int) int {
	index := len(height) - 1
	l1, l2, area := 0, index, 0

	for l1 < l2 {
		l := l2 - l1
		// 面积取决于短板
		// 长板往内移动时遇到更长的板没有用,取决于最短的板,此时长度还缩短了
		// 短板往内移动时可能遇到比当前长的短板,有机会面积增大
		if height[l1] > height[l2] {
			area = max(area, height[l2]*l)
			l2--
		} else {
			area = max(area, height[l1]*l)
			l1++
		}
	}
	return area
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```
