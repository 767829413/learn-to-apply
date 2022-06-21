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
func trap(height []int) int {
	l := len(height)
	sum := 0
	if l <= 0 {
		return sum
	}

	// 接水我们以当前位置是否能积水为参考标准
	// 当前位置高为h,左边最高设为lm,右边最高设为rm
	// 只有当前位置高度h小于min(lm,rm)的时候才能有积水等于 min(lm,rm) - h
	// 建立左右指针 lo hi,通过比较 height[lo] , height[hi]的高度确定当前位置依赖哪边高度
	lo, hi, lm, rm := 0, l-1, 0, 0
	for lo < hi {
		// 左边高度为依赖
		if height[lo] < height[hi] {
			// 判断lo位置是不是左边最高的
			if height[lo] >= lm {
				lm = height[lo]
			} else {
				// 不是最高则是可积水位置
				sum += lm - height[lo]
			}
			lo++
		} else {
			if height[hi] >= rm {
				rm = height[hi]
			} else {
				sum += rm - height[hi]
			}
			hi--
		}
	}
	return sum
}
```
