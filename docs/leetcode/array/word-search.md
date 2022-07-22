# 子集

## 题目

给定一个 `m x n` 二维字符网格 `board` 和一个字符串单词 `word` 。如果 `word` 存在于网格中，返回 `true` ；否则，返回 `false` 。

单词必须按照字母顺序，通过相邻的单元格内的字母构成，其中“相邻”单元格是那些水平相邻或垂直相邻的单元格。同一个单元格内的字母不允许被重复使用。

示例:

![word2.jpg](https://s2.loli.net/2022/07/21/EjJ7TNR6OvGqLab.jpg)

```text
输入：board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
输出：true
```

---

## code

```go
// Word search
func exist(board [][]byte, word string) bool {
	// {'A', 'B', 'C', 'E'},
	// {'S', 'F', 'C', 'S'},
	// {'A', 'D', 'E', 'E'},
	n := len(board)
	m := len(board[0])
	wordArr := []byte(word)
	// 构建map记录已走
	mr := make([][]int, n)
	for k := range mr {
		mr[k] = make([]int, m)
	}
	flag := make(chan bool)
	var dfs func(depth, row, col int, wordArr []byte)
	dfs = func(depth, row, col int, wordArr []byte) {
		// 标识存在
		if depth == len(wordArr)-1 {
			flag <- true
			return
		}

		// 向下走
		if row+1 < n && wordArr[depth+1] == board[row+1][col] && mr[row+1][col] == 0 {
			// fmt.Println("向下走", string(board[row+1][col]), string(wordArr[depth+1]), depth+1, mr[row+1][col])
			mr[row+1][col] = 1
			dfs(depth+1, row+1, col, wordArr)
			mr[row+1][col] = 0
		}

		// 向右走
		if col+1 < m && wordArr[depth+1] == board[row][col+1] && mr[row][col+1] == 0 {
			// fmt.Println("向右走", string(board[row][col+1]), string(wordArr[depth+1]), depth+1, mr[row][col+1])
			mr[row][col+1] = 1
			dfs(depth+1, row, col+1, wordArr)
			mr[row][col+1] = 0
		}

		// 向左走
		if col >= 1 && wordArr[depth+1] == board[row][col-1] && mr[row][col-1] == 0 {
			// fmt.Println("向左走", string(board[row][col-1]), string(wordArr[depth+1]), depth+1, mr[row][col-1])
			mr[row][col-1] = 1
			dfs(depth+1, row, col-1, wordArr)
			mr[row][col-1] = 0
		}

		// 向上走
		if row >= 1 && wordArr[depth+1] == board[row-1][col] && mr[row-1][col] == 0 {
			// fmt.Println("向上走", string(board[row-1][col]), string(wordArr[depth+1]), depth+1, mr[row-1][col])
			mr[row-1][col] = 1
			dfs(depth+1, row-1, col, wordArr)
			mr[row-1][col] = 0
		}
	}

	go func() {
		for i := 0; i < n; i++ {
			for j := 0; j < m; j++ {
				if wordArr[0] != board[i][j] {
					continue
				}
				mr[i][j] = 1
				dfs(0, i, j, wordArr)
				mr[i][j] = 0
			}
		}
		flag <- false
	}()

	select {
	case res := <-flag:
		return res
	case <-time.After(500 * time.Second):
		return false
	}
}
```
