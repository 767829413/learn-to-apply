# 接雨水

## 题目

给定 **n** 个非负整数表示每个宽度为 **1** 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

示例:

![示意图](https://pic.imgdb.cn/item/62b16774094754312956a906.png)

```text
输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
输出：6
解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 

输入：height = [4,2,0,3,2,5]
输出：9
```

---

## code

```go
// Trapping rain water
func trap(height []int) int {
	l := len(height)
	if l < 3 {
		return 0
	}
	// 接水我们以当前位置是否能积水为参考标准
	// 当前位置高为h,左边最高设为lm,右边最高设为rm
	// 只有当前位置高度h小于min(lm,rm)的时候才能有积水等于 min(lm,rm) - h
	// 建立左右指针 left right,首先通过比较 height[left] height[right]的高度确定接水的短板
	// 较短的一边指针位置与对应边界进行比较可以得出是否接住雨水
	left, right, lm, rm, sum := 0, len(height)-1, 0, 0, 0
	for left < right {
		// 接的雨水依赖右边
		if height[left] > height[right] {
			if rm > height[right] {
				sum += rm - height[right]
			} else {
				rm = height[right]
			}
			right--
		} else {
			if lm > height[left] {
				sum += lm - height[left]
			} else {
				lm = height[left]
			}
			left++
		}
	}
	return sum
}
```
