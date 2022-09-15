# 二叉树的右视图

## 题目

给定一个二叉树的 根节点 root，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

示例:

![tree.jpg](https://s2.loli.net/2022/09/15/QMHc6Z4v7fkqYm8.jpg)

```text
输入: [1,2,3,null,5,null,4]
输出: [1,3,4]
```

---

## code

```go
// Binary tree right side view
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func rightSideView(root *TreeNode) []int {
	if root == nil {
		return []int{}
	}
	var (
		res []int
		queue = []*TreeNode{root}
	)
	for len(queue) != 0 {
		l := len(queue)
		for i:=0;i<l;i++{
			node := queue[0]
			queue = queue[1:]
			if i == l-1 {
				res = append(res,node.Val )
			}
			if node.Left != nil {
				queue = append(queue, node.Left)
			}
			if node.Right != nil {
				queue = append(queue, node.Right)
			}
		}
	}
	return res
}

func NewTreeNode(arr []int) (*TreeNode, map[int]*TreeNode) {
	l, m := len(arr), make(map[int]*TreeNode, len(arr))
	var execBuild func(start int, arr []int) *TreeNode
	execBuild = func(start int, arr []int) *TreeNode {
		if start >= l || arr[start] == math.MinInt {
			return nil
		}
		node := &TreeNode{}
		node.Val = arr[start]
		node.Left = execBuild(start*2+1, arr)
		node.Right = execBuild(start*2+2, arr)
		m[start] = node
		return node
	}
	return execBuild(0, arr), m
}

func TestRightSideView(t *testing.T) {
	assert := assert.New(t)
	root, _ := NewTreeNode([]int{1, math.MinInt, 2})
	expected := []int{1, 2}
	assert.Equal(expected, rightSideView(root))
}
```
