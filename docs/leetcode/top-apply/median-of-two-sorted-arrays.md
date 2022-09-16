# 寻找两个正序数组的中位数

## 题目

给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的 中位数 。

算法的时间复杂度应该为 O(log (m+n)) 。

示例:

```text
输入：nums1 = [1,3], nums2 = [2]
输出：2.00000
解释：合并数组 = [1,2,3] ，中位数 2
```

---

## code

```go
// Median of two sorted arrays
// 时间复杂度O(m+n),凡人的解法
func findMedianSortedArrays(nums1 []int, nums2 []int) float64 {
	s1, l1, s2, l2 := 0, len(nums1), 0, len(nums2)
	l := l1 + l2
	if l == 0 {
		return 0
	}
	minLeft, minLeftV, minRight, minRightV, s, min := 0, -1, 0, -1, 0, l>>1
	if l%2 == 0 {
		minRight, minLeft = min, min-1
	} else {
		minRight, minLeft = min, min
	}

	for ; s1 < l1 && s2 < l2; s++ {
		val := 0
		if nums1[s1] < nums2[s2] {
			val = nums1[s1]
			s1++
		} else {
			val = nums2[s2]
			s2++
		}
		if s == minLeft {
			minLeftV = val
		}
		if s == minRight {
			minRightV = val
		}
	}
	if minLeftV == -1 || minRightV == -1 {
		var (
			i, j int
			num  []int
		)
		if s1 == l1 {
			i, j, num = s2, l2, nums2
		} else {
			i, j, num = s1, l1, nums1
		}
		for k := i; k < j; k++ {
			if s == minLeft {
				minLeftV = num[k]
			}
			if s == minRight {
				minRightV = num[k]
			}
			s++
		}
	}

	return float64(minLeftV+minRightV) / 2
}
```
