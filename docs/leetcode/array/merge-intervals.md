# 合并区间

## 题目

以数组 `intervals` 表示若干个区间的集合，其中单个区间为 `intervals[i] = [starti, endi]` 。请你合并所有重叠的区间，并返回 `一个不重叠的区间数组，该数组需恰好覆盖输入中的所有区间` 。

示例:

```text
输入：intervals = [[1,3],[2,6],[8,10],[15,18]]
输出：[[1,6],[8,10],[15,18]]
解释：区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].

输入：intervals = [[1,4],[4,5]]
输出：[[1,5]]
解释：区间 [1,4] 和 [4,5] 可被视为重叠区间。
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

func merge(intervals [][]int) [][]int {
	l := len(intervals)
	if l <= 1 {
		return intervals
	}
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][0] < intervals[j][0]
	})
	res, slow, quick := [][]int{intervals[0]}, 0, 1
	for quick < l {
		if res[slow][0] <= intervals[quick][0] && intervals[quick][0] <= res[slow][1] {
			var v []int
			if intervals[quick][1] > res[slow][1] {
				v = []int{res[slow][0], intervals[quick][1]}
			} else {
				v = []int{res[slow][0], res[slow][1]}
			}
			res[slow] = v
			quick++
		} else {
			res = append(res, intervals[quick])
			slow++
			quick++
		}
	}
	return res
}
```
