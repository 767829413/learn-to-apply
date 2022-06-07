# 搜索插入位置

## 题目

给定一个排序数组和一个目标值，在数组中找到目标值，并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。

请必须使用时间复杂度为 **O(log n)** 的算法。

示例:

```text
输入: nums = [1,3,5,6], target = 5
输出: 2

输入: nums = [1,3,5,6], target = 2
输出: 1
```

---

## code

```go
func searchInsert(nums []int, target int) int {
	l := len(nums)
	end := l - 1
	if l == 0 {
		return 0
	}
	if l == 1 {
		if target <= nums[0] {
			return 0
		} else if target > nums[0] {
			return 1
		}
	}
	// 单调且不重复可以直接判断头尾
	if target <= nums[0] {
		return 0
	}
	if target > nums[end] {
		return l
	} else if target == nums[end] {
		return end
	}
	lo, hi := 0, end
	// 区域分成[lo,mid],[mid+1,hi] lo = mid+1 | hi = mid
	for lo < hi {
		mid := (lo + hi) >> 1
		if target == nums[mid] {
			return mid
		} else if target > nums[mid] {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	// lo , hi 在找数组中不在的元素都是收缩边界,可以任选
	return hi
}
```
