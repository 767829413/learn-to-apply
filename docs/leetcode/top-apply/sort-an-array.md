# 排序数组

## 题目

给你一个整数数组 nums，请你将该数组升序排列。

示例:

```text
输入：nums = [5,2,3,1]
输出：[1,2,3,5]

输入：nums = [5,1,1,2,0,0]
输出：[0,0,1,1,2,5]
```

---

## code

```go
// Sort an array
func sortArray(nums []int) []int {
	// 已排序检查
	if isSorted(nums) {
		return nums
	}

	if false {
		// 快排
		position := func(nums []int, i, j int) int {
			pre, posVal := i, nums[j]
			for l := i; l < j; l++ {
				if nums[l] < posVal {
					nums[pre], nums[l] = nums[l], nums[pre]
					pre++
				}
			}
			nums[pre], nums[j] = nums[j], nums[pre]
			return pre
		}
		var quickSortPart func(nums []int, i, j int)
		quickSortPart = func(nums []int, i, j int) {
			if i >= j {
				return
			}
			pos := position(nums, i, j)
			quickSortPart(nums, i, pos-1)
			quickSortPart(nums, pos+1, j)
		}
		quickSort := func(nums []int) {
			l := len(nums)
			end := l - 1
			quickSortPart(nums, 0, end)
		}
		quickSort(nums)
	} else {
		merge := func(nums []int, i, mid, j int) {
			tmp := make([]int, j-i+1)
			// 合并区间 [i,mid] [mid+1,j]
			start, l, r := 0, i, mid+1
			for ; l <= mid && r <= j; start++ {
				if nums[l] <= nums[r] {
					tmp[start] = nums[l]
					l++
				} else {
					tmp[start] = nums[r]
					r++
				}
			}
			s, e := r, j
			if l <= mid {
				s, e = l, mid
			}
			for k := s; k <= e; k, start = k+1, start+1 {
				tmp[start] = nums[k]
			}

			for k, index := i, 0; k <= j && index < len(tmp); k, index = k+1, index+1 {
				nums[k] = tmp[index]
			}
		}
		var mergeSortPart func(nums []int, i, j int)
		mergeSortPart = func(nums []int, i, j int) {
			//递归终止条件
			if i >= j {
				return
			}
			mid := (i + j) >> 1
			mergeSortPart(nums, i, mid)
			mergeSortPart(nums, mid+1, j)
			merge(nums, i, mid, j)
		}
		// 归并
		mergeSort := func(nums []int) {
			l := len(nums)
			end := l - 1
			mergeSortPart(nums, 0, end)
		}
		mergeSort(nums)
	}

	return nums
}

// 已排序检查
func isSorted(arr []int) bool {
	for i := 1; i < len(arr); i++ {
		if arr[i] < arr[i-1] {
			return false
		}
	}
	return true
}
```
