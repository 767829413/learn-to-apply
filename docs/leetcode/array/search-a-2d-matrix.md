# 搜索二维矩阵

## 题目

编写一个高效的算法来判断 `m x n` 矩阵中，是否存在一个目标值。该矩阵具有如下特性：

* 每行中的整数从左到右按升序排列。
* 每行的第一个整数大于前一行的最后一个整数。

示例:

![mat.jpg](https://s2.loli.net/2022/07/18/FCABP3OLkl6zDHu.jpg)

```text
输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3
输出：true
```

---

## code

```go
// Search a 2d matrix
func searchMatrix(matrix [][]int, target int) bool {
	n := len(matrix)
	m := len(matrix[0])
	if n == 0 || m == 0 || target < matrix[0][0] || target > matrix[n-1][m-1] {
		return false
	}
	// 二分查找
	secondPartSearch := func(a []int, target int) bool {
		lo, hi := 0, len(a)-1
		if lo == hi {
			return a[lo] == target
		}
		// 定义[lo,mid],[mid+1,hi]区间,lo = mid+1 | hi = mid
		for lo <= hi {
			mid := (lo + hi) >> 1
			if target < a[mid] {
				hi = mid - 1
			} else if target > a[mid] {
				lo = mid + 1
			} else {
				return true
			}
		}
		return false
	}

	// 对第一列进行二分查找一个稍微大一点的值
	row, rowL, rowH := -1, 0, n-1
	for rowL <= rowH {
		mid := (rowL + rowH) >> 1
		if target <= matrix[mid][m-1] {
			row = mid
			rowH = mid - 1
		} else if target > matrix[mid][m-1] {
			rowL = mid + 1
		}
	}
	if row == -1 {
		return false
	}

	return secondPartSearch(matrix[row], target)
}
```
