# 颜色分类

## 题目

给定一个包含红色、白色和蓝色、共 `n` 个元素的数组 `nums` ，`原地`对它们进行排序，使得相同颜色的元素相邻，并按照红色、白色、蓝色顺序排列。

我们使用整数 `0`、 `1` 和 `2` 分别表示红色、白色和蓝色。

必须在不使用库的sort函数的情况下解决这个问题。

示例:

```text
输入：nums = [2,0,2,1,1,0]
输出：[0,0,1,1,2,2]

输入：nums = [2,0,1]
输出：[0,1,2]
```

---

## code

```go
// Sort colors
func sortColors(nums []int) {
	// 整数 0、 1 和 2 分别表示红色、白色和蓝色
	// 按照红色、白色、蓝色顺序排列
	start, zero, two := 0, 0, len(nums)-1

	// 构建区间 [0,zero) [zero,two] (two,len(nums)-1]
	for start <= two {
		if nums[start] == 0 {
			nums[start], nums[zero] = nums[zero], nums[start]
			zero++
			start++
		} else if nums[start] == 1 {
			start++
		} else {
			nums[start], nums[two] = nums[two], nums[start]
			two--
		}
		// fmt.Println(start, zero, two, nums)
	}
}
```
