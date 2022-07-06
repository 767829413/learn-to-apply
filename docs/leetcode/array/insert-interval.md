# 插入区间

## 题目

给你一个 `无重叠的` ，按照区间起始端点排序的区间列表。

在列表中插入一个新的区间，你需要确保列表中的区间仍然有序且不重叠（如果有必要的话，可以合并区间）。

示例:

```text
输入：intervals = [[1,3],[6,9]], newInterval = [2,5]
输出：[[1,5],[6,9]]

输入：intervals = [[1,2],[3,5],[6,7],[8,10],[12,16]], newInterval = [4,8]
输出：[[1,2],[3,10],[12,16]]
解释：这是因为新的区间 [4,8] 与 [3,5],[6,7],[8,10] 重叠。

输入：intervals = [], newInterval = [5,7]
输出：[[5,7]]

输入：intervals = [[1,5]], newInterval = [2,3]
输出：[[1,5]]

输入：intervals = [[1,5]], newInterval = [2,7]
输出：[[1,7]]
```

---

## code

```go
// Merge intervals
func merge(intervals [][]int) [][]int {
	box := [][]int{}
	l := len(intervals)
	if l == 1 {
		return intervals
	}
	max := func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	// 首先对每个区间的起始位置进行排序
	sort.Slice(intervals, func(i int, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})
	// 构建一个对比的序列
	box = append(box, intervals[0])
	for i := 1; i < l; i++ {
		j := len(box) - 1
		// 如果a,b区间能合并,必然是 a区间[i,j]中的j >= b区间[x,y]中x
		if intervals[i][0] <= box[j][1] {
			box[j][1] = max(intervals[i][1], box[j][1])
		} else { // 不能合并那就得把当前值push到对比序列里
			box = append(box, intervals[i])
		}
	}
	return box
}

// Insert interval
func insert(intervals [][]int, newInterval []int) [][]int {
	l := len(intervals)
	if l == 0 {
		return [][]int{newInterval}
	}
	intervals = append(intervals, newInterval)

	return merge(intervals)
}
```
