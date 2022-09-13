# 二分查找

## 题目

给定一个 n 个元素有序的（升序）整型数组 nums 和一个目标值 target  ，写一个函数搜索 nums 中的 target，如果目标值存在返回下标，否则返回 -1。

示例:

```text
输入: nums = [-1,0,3,5,9,12], target = 9
输出: 4
解释: 9 出现在 nums 中并且下标为 4
```

---

## code

```go
// Binary search
func search(nums []int, target int) int {
	low, hig := 0, len(nums)-1
	for low <= hig {
		mid := (low + hig) >> 1
		if target > nums[mid] {
			low = mid + 1
		} else if target < nums[mid] {
			hig = mid - 1
		} else {
			return mid
		}
	}
	return -1
}
```
