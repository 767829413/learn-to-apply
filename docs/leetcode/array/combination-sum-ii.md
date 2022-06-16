# 组合总和 II

## 题目

给定一个候选人编号的集合 **candidates** 和一个目标数 **target** ，找出 **candidates** 中所有可以使数字和为 **target** 的组合。

**candidates** 中的每个数字在每个组合中只能使用 **一次** 。

注意：解集不能包含重复的组合。

示例:

```text
输入: candidates = [10,1,2,7,6,1,5], target = 8,
输出:
[
[1,1,6],
[1,2,5],
[1,7],
[2,6]
]

输入: candidates = [2,5,2,1,2], target = 5,
输出:
[
[1,2,2],
[5]
]
```

---

## code

```go
func combinationSum2(candidates []int, target int) [][]int {
	box := [][]int{}
	l := len(candidates)
	if l <= 0 {
		return box
	}
	// 排序,加速剪枝判断
	sort.Ints(candidates)
	// 采用 选择 | 不选择 来构建递归树
	// 构建hash, [pos][0] [pos][1] 表示 candidates 某个数的值和次数
	// var m [][2]int
	// for _, n := range candidates {
	// 	if m == nil || n != m[len(m)-1][0] {
	// 		m = append(m, [2]int{n, 1})
	// 	} else {
	// 		m[len(m)-1][1]++
	// 	}
	// }
	// min := func(a, b int) int {
	// 	if a > b {
	// 		return b
	// 	}
	// 	return a
	// }
	// var dfs func(target, idx int, tmp []int)
	// dfs = func(target, idx int, tmp []int) {
	// 	if target <= 0 || idx == l || target < m[idx][0] {
	// 		if target == 0 {
	// 			newTmp := make([]int, len(tmp))
	// 			copy(newTmp, tmp)
	// 			box = append(box, newTmp)
	// 		}
	// 		return
	// 	}

	// 	dfs(target, idx+1, tmp) // 不选择

	// 	repeat := min(target/m[idx][0], m[idx][1]) // 通过用 目标值/当前位置的值 和 次数 判断,选取小的重复递归

	// 	for i := 1; i <= repeat; i++ {// 选择
	// 		tmp = append(tmp, m[idx][0])
	// 		dfs(target-i*m[idx][0], idx+1, tmp)
	// 	}
	// 	tmp = tmp[:len(tmp)-repeat]
	// }
	// dfs(target, 0, []int{})
	var dfs func(target, start int, tmp []int)
	dfs = func(target, start int, tmp []int) {
		if target <= 0 {
			if target == 0 {
				newTmp := make([]int, len(tmp))
				copy(newTmp, tmp)
				box = append(box, newTmp)
			}
			return
		}
		for i := start; i < l; i++ {
			v := target - candidates[i]
			if v < 0 {
				break
			}
			// 同层相同必须跳过,上下层相同可以
			if start < i && candidates[i] == candidates[i-1] {
				continue
			}
			tmp = append(tmp, candidates[i])
			// 已经使用过的不在使用
			dfs(v, i+1, tmp)
			tmp = tmp[:len(tmp)-1]
		}
	}
	dfs(target, 0, []int{})
	return box
}
```
