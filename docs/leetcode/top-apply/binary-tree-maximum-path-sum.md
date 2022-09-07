# 二叉树中的最大路径和

## 题目

`路径` 被定义为一条从树中任意节点出发，沿父节点-子节点连接，达到任意节点的序列。同一个节点在一条路径序列中 `至多出现一次` 。该路径 `至少包含一个` 节点，且不一定经过根节点。

`路径和` 是路径中各节点值的总和。

给你一个二叉树的根节点 `root` ，返回其 最大路径和 。

示例:

![exx2.jpg](https://s2.loli.net/2022/09/07/XM6LHekPN2fI5bl.jpg)

```text
输入：root = [-10,9,20,null,null,15,7]
输出：42
解释：最优路径是 15 -> 20 -> 7 ，路径和为 15 + 20 + 7 = 42
```

---

## code

```go
// Binary tree maximum path sum
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func maxPathSum(root *TreeNode) int {
	var dfs func(node *TreeNode) int
	sum := math.MinInt
	max := func(a, b int) int {
		if a > b {
			return a
		}
		return b
	}
	dfs = func(node *TreeNode) int {
		if node == nil {
			return 0
		}
		left := dfs(node.Left)
		right := dfs(node.Right)
		// 当前节点具有的最大路径和(包括子树)
		tmpNum := left + right + node.Val
		// 与已知节点的路径和比较,选取最大
		sum = max(tmpNum, sum)
		// 输出的最大路径和,从当前根节点下来,只能走左或走右,不能走向左边后又走向右边
		return max(max(left, right)+node.Val, 0)
	}
	dfs(root)
	return sum
}

func NewTreeNode(arr []int) (*TreeNode, map[int]*TreeNode) {
	l, m := len(arr), make(map[int]*TreeNode, len(arr))
	var execBuild func(start int, arr []int) *TreeNode
	execBuild = func(start int, arr []int) *TreeNode {
		if start >= l || arr[start] == -1 {
			return nil
		}
		node := &TreeNode{
			Val:   arr[start],
			Left:  execBuild(start*2+1, arr),
			Right: execBuild(start*2+2, arr),
		}
		m[start] = node
		return node
	}
	return execBuild(0, arr), m
}

func TestMaxPathSum(t *testing.T) {
	assert := assert.New(t)
	root, _ := NewTreeNode([]int{-10, 9, 20, -1, -1, 15, 7})
	expected := 42
	assert.Equal(expected, maxPathSum(root))
}
```
