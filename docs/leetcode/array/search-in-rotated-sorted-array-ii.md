# 搜索旋转排序数组 II

## 题目

已知存在一个按非降序排列的整数数组 `nums` ，数组中的值不必互不相同。

在传递给函数之前，`nums` 在预先未知的某个下标 `k（0 <= k < nums.length）`上进行了 旋转 ，使数组变为 `[nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]（下标 从 0 开始 计数）`。例如， `[0,1,2,4,4,4,5,6,6,7]` 在下标 5 处经旋转后可能变为 `[4,5,6,6,7,0,1,2,4,4]` 。

给你 `旋转后` 的数组 `nums` 和一个整数 `target` ，请你编写一个函数来判断给定的目标值是否存在于数组中。如果 `nums` 中存在这个目标值 `target` ，则返回 `true` ，否则返回 `false` 。

你必须尽可能减少整个操作步骤。

示例:

```text
输入：nums = [2,5,6,0,0,1,2], target = 0
输出：true

输入：nums = [2,5,6,0,0,1,2], target = 3
输出：false
```

---

## code

```go
// Search in rotated sorted array ii
func search(nums []int, target int) bool {
	l := len(nums)
	if l == 0 {
		return false
	}
	if l == 1 {
		return nums[0] == target
	}
	for low, hig := 0, l-1; low <= hig; {
		mid := (low + hig) >> 1
		// fmt.Println(low, hig, mid)
		// fmt.Println(nums[low], nums[hig], nums[mid])
		if target == nums[mid] || target == nums[low] || target == nums[hig] {
			return true
		}

		if nums[mid] == nums[low] {
			low++
			continue
		}
		if nums[mid] == nums[hig] {
			hig--
			continue
		}

		// 左边有序
		if nums[mid] > nums[low] {
			// 在有序部分
			if nums[low] < target && target < nums[mid] {
				hig = mid - 1
			} else {
				low = mid + 1
			}

		} else {
			// 在有序部分
			if nums[hig] > target && target > nums[mid] {
				low = mid + 1
			} else {
				hig = mid - 1
			}
		}
	}
	return false
}
```
