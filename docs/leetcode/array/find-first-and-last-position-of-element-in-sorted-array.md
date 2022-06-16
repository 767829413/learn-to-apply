# 在排序数组中查找元素的第一个和最后一个位置

## 题目

给定一个按照升序排列的整数数组 **nums**，和一个目标值 **target** 。找出给定目标值在数组中的开始位置和结束位置。

如果数组中不存在目标值 **target** ，返回 **[-1, -1]**。

你可以设计并实现时间复杂度为 **O(log n)** 的算法解决此问题吗？

示例:

```text
输入：nums = [5,7,7,8,8,10], target = 8
输出：[3,4]
```

---

## code

```go
func searchRange(nums []int, target int) []int {
	l := len(nums)
	if l == 0 {
		return []int{-1, -1}
	}
	if l == 1 {
		if nums[0] == target {
			return []int{0, 0}
		} else {
			return []int{-1, -1}
		}
	}
	lo, hi, L, R := 0, l-1, 0, 0
	// 定义[lo,mid],[mid+1,hi]区间,lo = mid+1 | hi = mid
	// target <= nums[mid] hi不断向左边界逼近,最终值为最左边第一个出现的target位置
	for lo < hi {
		mid := (lo + hi) >> 1
		if target <= nums[mid] {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	if nums[hi] != target {
		return []int{-1, -1}
	}
	L = hi
	lo, hi = 0, l-1
	// 定义[lo,mid-1],[mid,hi]区间,lo = mid | hi = mid-1
	// target >= nums[mid] lo不断向右边界逼近,最终值为最右边最后一个出现的target位置
	for lo < hi {
		mid := (lo + hi + 1) >> 1
		if target >= nums[mid] {
			lo = mid
		} else {
			hi = mid - 1
		}
	}
	R = lo
	return []int{L, R}
}
```
