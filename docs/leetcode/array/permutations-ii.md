# 全排列 II

## 题目

给定一个可包含重复数字的序列 **nums** ，按任意顺序 返回所有不重复的全排列。

示例:

```text
输入：nums = [1,1,2]
输出：
[[1,1,2],
 [1,2,1],
 [2,1,1]]

输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```

---

## code

```go
func permuteUnique(nums []int) [][]int {
	box := [][]int{}
	l := len(nums)
	if l == 0 {
		return box
	}
	// 排序,方便剪枝
	sort.Ints(nums)
	var dfs func(depth int, path []int, used []bool)
	dfs = func(depth int, path []int, used []bool) {
		if depth == l {
			//fmt.Println(path)
			newPath := make([]int, len(path))
			copy(newPath, path)
			box = append(box, newPath)
			return
		}
		for i := 0; i < l; i++ {
			// 同深度去重的关键是上一层同值不能被使用,被使用了当前同值不可跳过
			// 剩下的就是普通的回溯了
			if i > 0 && nums[i-1] == nums[i] && !used[i-1] {
				continue
			}
			if used[i] {
				continue
			}
			path = append(path, nums[i])
			used[i] = true
			dfs(depth+1, path, used)
			used[i] = false
			path = path[:len(path)-1]
		}
	}
	dfs(0, []int{}, make([]bool, l))
	return box
}
```
