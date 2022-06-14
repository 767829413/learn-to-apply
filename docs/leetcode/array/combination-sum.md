# 搜索插入位置

## 题目

给你一个 无重复元素 的整数数组 **candidates** 和一个目标整数 **target** ，找出 **candidates** 中可以使数字和为目标数 **target** 的 **所有** 不同组合 ，并以列表形式返回。你可以按 **任意顺序** 返回这些组合。

**candidates** 中的 **同一个** 数字可以 **无限制重复被选取** 。如果至少一个数字的被选数量不同，则两种组合是不同的。

对于给定的输入，保证和为 **target** 的不同组合数少于 150 个。

示例:

```text
输入：candidates = [2,3,6,7], target = 7
输出：[[2,2,3],[7]]
解释：2 和 3 可以形成一组候选，2 + 2 + 3 = 7 。注意 2 可以使用多次。
7 也是一个候选， 7 = 7 。
仅有这两种组合。

输入: candidates = [2,3,5], target = 8
输出: [[2,2,2,2],[2,3,3],[3,5]]

输入: candidates = [2], target = 1
输出: []
```

---

## code

```go
func combinationSum(candidates []int, target int) [][]int {
	l := len(candidates)
	boxs := [][]int{}
	if l <= 0 {
		return boxs
	}
	// 排序,方便减枝
	sort.Ints(candidates)
	// 采用 选择 | 不选择 来构建递归树
	// var dfs func(target, idx int, tmp []int)
	// dfs = func(target, idx int, tmp []int) {
	// 	if target <= 0 || idx == l {
	// 		if target == 0 {
	// 			newTmp := make([]int, len(tmp))
	// 			copy(newTmp, tmp)
	// 			boxs = append(boxs, newTmp)
	// 		}
	// 		return
	// 	}

	// 	// 不选择
	// 	dfs(target, idx+1, tmp)
	// 	// 选择
	// 	v := target - candidates[idx]
	// 	if v >= 0 {
	// 		tmp = append(tmp, candidates[idx])
	// 		dfs(v, idx, tmp)
	// 		tmp = tmp[:len(tmp)-1]
	// 	}
	// }
	// dfs(target, 0, []int{})
	// 采用目标值减值构建递归树
	var dfs func(target, start int, tmp []int)
	dfs = func(target, start int, tmp []int) {
		if target <= 0 {
			if target == 0 {
				newTmp := make([]int, len(tmp))
				copy(newTmp, tmp)
				boxs = append(boxs, newTmp)
			}
			return
		}
		// 这里保证同层第二个节点不使用第一个节点在 candidates 里完全使用过的值
		// candidates = [2,3,5]
		// 第一个节点是(-2 -3 -5) 第二个节点(-3 -5)
		for i := start; i < l; i++ {
			v := target - candidates[i]
			if v < 0 {
				break
			}
			tmp = append(tmp, candidates[i])
			dfs(v, i, tmp)
			tmp = tmp[:len(tmp)-1]
		}
	}
	dfs(target, 0, []int{})
	return boxs
}
```
