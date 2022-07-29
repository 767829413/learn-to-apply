# 子集 II

## 题目

给你一个整数数组 `nums` ，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）。

解集 `不能` 包含重复的子集。返回的解集中，子集可以按 `任意顺序` 排列。

示例:

```text
输入：nums = [1,2,2]
输出：[[],[1],[1,2],[1,2,2],[2],[2,2]]

输入：nums = [0]
输出：[[],[0]]
```

---

## code

```go
// Subsets ii
func subsetsWithDup(nums []int) [][]int {
	box := [][]int{}
	l := len(nums)
	if l == 0 {
		return append(box, []int{})
	}
	if l == 1 {
		return append(append(box, []int{}), nums)
	}
	var dfs func(start int, tmp []int)
	dfs = func(start int, tmp []int) {
		newTmp := make([]int, len(tmp))
		copy(newTmp, tmp)
		box = append(box, newTmp)
		for i := start; i < l; i++ {
			if i > start && nums[i] == nums[i-1] {
				continue
			}
			tmp = append(tmp, nums[i])
			// 易错点,同树枝上可以重复选择
			// 同层不可以重复
			dfs(i+1, tmp)
			tmp = tmp[:len(tmp)-1]
		}
	}
	// 排序,加速剪枝
	sort.Ints(nums)
	dfs(0, []int{})
	return box
}
```
