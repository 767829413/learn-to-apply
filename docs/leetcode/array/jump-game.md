# 跳跃游戏

## 题目

给定一个非负整数数组 `nums` ，你最初位于数组的 `第一个下标` 。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

判断你是否能够到达最后一个下标。

示例:

```text
输入：nums = [2,3,1,1,4]
输出：true
解释：可以先跳 1 步，从下标 0 到达下标 1, 然后再从下标 1 跳 3 步到达最后一个下标。

输入：nums = [3,2,1,0,4]
输出：false
解释：无论怎样，总会到达下标为 3 的位置。但该下标的最大跳跃长度是 0 ， 所以永远不可能到达最后一个下标。
```

---

## code

```go
func canJump(nums []int) bool {
	l := len(nums)
	if l != 1 && nums[0] == 0 {
		return false
	}
	if l == 1 || nums[0] >= l-1 {
		return true
	}

	// 反向找可以跳的位置, O(n²)
	// end, find := l-1, false
	// for {
	// 	find = false
	// 	for i := 0; i < end; i++ {
	// 		if i+nums[i] >= end {
	// 			end = i
	// 			find = true
	// 		}
	// 	}
	// 	if end == 0 {
	// 		return true
	// 	}
	// 	// fmt.Println(find, end)
	// 	if !find {
	// 		return false
	// 	}
	// }

	// 维持连续正向跳最远 O(n)
	// 1. 如果某一个作为 起跳点 的格子可以跳跃的距离是 3，那么表示后面 3 个格子都可以作为 起跳点
	// 2. 可以对每一个能作为 起跳点 的格子都尝试跳一次，把 能跳到最远的距离 不断更新
	// 3. 如果可以一直跳到最后，就成功了
	index, max := 0, func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	for i := 0; i < l; i++ {
		// 当前位置已经大于前面最大能跳的位置了那必然是无法继续跳下去了
		if i > index {
			return false
		}
		index = max(index, nums[i]+i)
	}
	return true
}
```
