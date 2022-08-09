# 三数之和

## 题目

给你一个包含 `n` 个整数的数组 `nums`，判断 `nums` 中是否存在三个元素 `a，b，c` ，使得 `a + b + c = 0 ？`请你找出所有和为 `0` 且不重复的三元组。

注意：答案中不可以包含重复的三元组。

示例:

```text
输入：nums = [-1,0,1,2,-1,-4]
输出：[[-1,-1,2],[-1,0,1]]

输入：nums = []
输出：[]

输入：nums = [0]
输出：[]
```

---

## code

```go
// 3sum
func threeSum(nums []int) [][]int {
	l, res := len(nums), [][]int{}
	if l <= 3 {
		if l == 3 && nums[0]+nums[1]+nums[2] == 0 {
			return append(res, nums)
		}
		return res
	}

	// 排序,加速判断
	sort.Ints(nums)
	// 遍历元素
	for i := range nums {
		// 利用排序的结果快速过滤
		if nums[i] > 0 {
			break
		}
		// 过滤重复
		if i-1 >= 0 && nums[i-1] == nums[i] {
			continue
		}
		//准备两个指针来组合结果
		lo, hi := i+1, l-1
		for lo < hi {
			r := nums[i] + nums[lo] + nums[hi]
			if r == 0 {
				res = append(res, []int{nums[i], nums[lo], nums[hi]})
				// 过滤重复
				for lo < hi && nums[lo+1] == nums[lo] {
					lo++
				}
				// 过滤重复
				for lo < hi && nums[hi-1] == nums[hi] {
					hi--
				}
				lo++
				hi--
			} else if r > 0 {
				// 过滤重复
				for lo < hi && nums[hi-1] == nums[hi] {
					hi--
				}
				hi--
			} else {
				// 过滤重复
				for lo < hi && nums[lo+1] == nums[lo] {
					lo++
				}
				lo++
			}
		}
	}
	return res
}
```
