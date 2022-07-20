# 子集

## 题目

给你一个整数数组 `nums` ，数组中的元素 `互不相同` 。返回该数组所有可能的`子集（幂集）`。

解集 `不能` 包含重复的子集。你可以按 `任意顺序` 返回解集。

示例:

```text
输入：nums = [1,2,3]
输出：[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

输入：nums = [0]
输出：[[],[0]]
```

---

## code

```go
// Subsets
func subsets(nums []int) [][]int {
	// 首先画出递归参照图
	l := len(nums)
	if l == 0 {
		return append([][]int{}, []int{})
	}
	if l == 1 {
		return append([][]int{{}}, nums)
	}
	box := [][]int{}
	var dfs func(start int, tmp []int)
	dfs = func(start int, tmp []int) {
		newTmp := make([]int, len(tmp))
		copy(newTmp, tmp)
		box = append(box, newTmp)
		for i := start; i < l; i++ {
			tmp = append(tmp, nums[i])
			// 当前递归 return 时候,应该是i+1而不是 start+1 组合排列是有区别的
			dfs(i+1, tmp)
			tmp = tmp[:len(tmp)-1]
		}
	}
	dfs(0, []int{})
	return box
}
```
