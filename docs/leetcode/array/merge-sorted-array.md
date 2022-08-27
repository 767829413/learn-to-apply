# 合并两个有序数组

## 题目

给你两个按 `非递减顺序` 排列的整数数组 `nums1` 和 `nums2`，另有两个整数 `m` 和 `n` ，分别表示 `nums1` 和 `nums2` 中的元素数目。

请你 `合并 nums2 到 nums1` 中，使合并后的数组同样按 `非递减顺序` 排列。

注意：最终，合并后数组不应由函数返回，而是存储在数组 `nums1` 中。为了应对这种情况，`nums1` 的初始长度为 `m + n`，其中前 `m` 个元素表示应合并的元素，后 `n` 个元素为 `0` ，应忽略。`nums2` 的长度为 `n` 。

示例:

```text
输入：nums1 = [1,2,3,0,0,0], m = 3, nums2 = [2,5,6], n = 3
输出：[1,2,2,3,5,6]
解释：需要合并 [1,2,3] 和 [2,5,6] 。
合并结果是 [1,2,2,3,5,6] ，其中斜体加粗标注的为 nums1 中的元素。

输入：nums1 = [1], m = 1, nums2 = [], n = 0
输出：[1]
解释：需要合并 [1] 和 [] 。
合并结果是 [1] 。

输入：nums1 = [0], m = 0, nums2 = [1], n = 1
输出：[1]
解释：需要合并的数组是 [] 和 [1] 。
合并结果是 [1] 。
注意，因为 m = 0 ，所以 nums1 中没有元素。nums1 中仅存的 0 仅仅是为了确保合并结果可以顺利存放到 nums1 中。
```

---

## code

```go
// Merge sorted array
func merge1(nums1 []int, m int, nums2 []int, n int) {
	box := []int{}
	i, j := 0, 0
	for i < m && j < n {
		if nums2[j] >= nums1[i] {
			box = append(box, nums1[i])
			i++
		} else {
			box = append(box, nums2[j])
			j++
		}
	}
	if i == m {
		for ; j < n; j++ {
			box = append(box, nums2[j])
		}
	}
	if j == n {
		for ; i < m; i++ {
			box = append(box, nums1[i])
		}
	}
	copy(nums1, box)
}

func merge2(nums1 []int, m int, nums2 []int, n int) {
	// 因为都是有序数组,且 nums1 是放合并数组后的结果
	// 可以从大到小进行比较,一次放在 nums1 右边
	// 如果 nums1 有效长度遍历完,那就直接倒插入 nums2 剩余数值
	// 如果 nums2 有效长度遍历完,那么 nums1 就是已经合并的结果,直接 返回
	right, i, j := m+n-1, m-1, n-1
	for ; i >= 0 && j >= 0; right-- {
		if nums2[j] >= nums1[i] {
			nums1[right] = nums2[j]
			j--
		} else {
			nums1[i], nums1[right] = nums1[right], nums1[i]
			i--
		}
	}
	if i == -1 {
		for s := j; s >= 0; s-- {
			nums1[right] = nums2[s]
			right--
		}
	}
}
```
