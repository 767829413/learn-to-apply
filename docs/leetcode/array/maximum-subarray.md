# 搜索插入位置

## 题目

给你一个整数数组 `nums` ，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

`子数组` 是数组中的一个连续部分。

示例:

```text
输入：nums = [-2,1,-3,4,-1,2,1,-5,4]
输出：6
解释：连续子数组 [4,-1,2,1] 的和最大，为 6 。

输入：nums = [1]
输出：1

输入：nums = [5,4,-1,7,8]
输出：23
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
