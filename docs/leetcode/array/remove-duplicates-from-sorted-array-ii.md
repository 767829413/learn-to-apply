# 删除有序数组中的重复项 II

## 题目

给你一个有序数组 `nums` ，请你 原地 删除重复出现的元素，使得出现次数超过两次的元素`只出现两次` ，返回删除后数组的新长度。

不要使用额外的数组空间，你必须在 `原地 修改输入数组` 并在使用 `O(1)` 额外空间的条件下完成。

示例:

```text
输入：nums = [1,1,1,2,2,3]
输出：5, nums = [1,1,2,2,3]
解释：函数应返回新长度 length = 5, 并且原数组的前五个元素被修改为 1, 1, 2, 2, 3 。 不需要考虑数组中超出新长度后面的元素。

输入：nums = [0,0,1,1,1,1,2,3,3]
输出：7, nums = [0,0,1,1,2,3,3]
解释：函数应返回新长度 length = 7, 并且原数组的前五个元素被修改为 0, 0, 1, 1, 2, 3, 3 。 不需要考虑数组中超出新长度后面的元素。
```

---

## code

```go
// Remove duplicates from sorted array ii
func removeDuplicates(nums []int) int {
	i, j := 0, 1
	for j < len(nums) {
		if nums[i] == nums[j] {
			if j < len(nums) && j > i+1 {
				nums = append(nums[:j], nums[j+1:]...)
			} else {
				j++
			}
		} else {
			i = j
			j += 1
		}
	}
	return j
}
```
