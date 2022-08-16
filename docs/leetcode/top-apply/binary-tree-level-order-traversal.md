# 二叉树的层序遍历

## 题目

给你二叉树的根节点 `root` ，返回其节点值的 `层序遍历` 。 （即逐层地，从左到右访问所有节点）。

示例:

![tree1.jpg](https://s2.loli.net/2022/08/16/oj5W1Sd8trvnsFh.jpg)

```text
输入：root = [3,9,20,null,null,15,7]
输出：[[3],[9,20],[15,7]]

输入：root = [1]
输出：[[1]]

输入：root = []
输出：[]
```

---

## code

```go
// Binary tree level order traversal
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func levelOrder(root *TreeNode) [][]int {
	res := [][]int{}
	if root == nil {
		return res
	}
	// 使用BFS进行解答
	// 构造队列queue,用来调整层级数据
	bfs := func(queue []*TreeNode) {
		for len(queue) > 0 {
			level := []int{}
			// 记录操作前队列的长度,方便划分层级
			l := len(queue)
			// 遍历原队列长度
			for i := 0; i < l; i++ {
				// 拿出头节点
				node := queue[0]
				// 出队
				queue = queue[1:]
				level = append(level, node.Val)
				if node.Left != nil {
					// 入队
					queue = append(queue, node.Left)
				}
				if node.Right != nil {
					// 入队
					queue = append(queue, node.Right)
				}
			}
			// 遍历结束就是需要的结果
			res = append(res, level)
		}
	}
	bfs([]*TreeNode{root})
	return res
}

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

// 用-1替代null
func NewTreeNode(arr []int) *TreeNode {
	l := len(arr)
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
		return node
	}
	return execBuild(0, arr)
}
```
