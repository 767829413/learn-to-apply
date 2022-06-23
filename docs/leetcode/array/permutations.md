# 全排列

## 题目

给定一个不含重复数字的数组 **nums** ，返回其 **所有可能的全排列** 。你可以 **按任意顺序** 返回答案。

示例:

```text
输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

输入：nums = [0,1]
输出：[[0,1],[1,0]]

输入：nums = [1]
输出：[[1]]
```

---

## code

```go
func permute(nums []int) [][]int {
	box := [][]int{}
	l := len(nums)
	if l == 0 {
		return box
	}
	var dfs func(depth int, used []bool, path []int)
	// 利用深度进行逐层循环遍历
	dfs = func(depth int, used []bool, path []int) {
		// 设置终止条件,深度
		if depth == l {
			// 必须用新的值去拷贝,不然数据会被重复设置
			// 切片底层还是数组,引用导致的
			newPath := make([]int, len(path))
			copy(newPath, path)
			box = append(box, newPath)
			return
		}
		for i := 0; i < l; i++ {
			// 剔除重复使用
			if used[i] {
				continue
			}
			// 开始尝试状态
			path = append(path, nums[i])
			used[i] = true
			dfs(depth+1, used, path)
			// 恢复到尝试前的状态
			path = path[:len(path)-1]
			used[i] = false
		}
	}
	dfs(0, make([]bool, l), []int{})
	return box
}
```
