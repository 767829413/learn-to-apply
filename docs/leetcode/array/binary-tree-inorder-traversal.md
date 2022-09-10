# 二叉树的中序遍历

## 题目

给定一个二叉树的根节点 root ，返回 它的 中序 遍历 。

示例:

![inorder_1.jpg](https://s2.loli.net/2022/09/10/xKdCn37DzbhNcLS.jpg)

```text
输入：root = [1, null, 2, null, null, 3]
输出：[1,3,2]
```

---

## code

```go
// Binary tree inorder traversal
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func inorderTraversal(root *TreeNode) []int {
	var (
		ergodic func(node *TreeNode)
		res     []int
	)
	ergodic = func(node *TreeNode) {
		if node == nil {
			return
		}
		ergodic(node.Left)
		res = append(res, node.Val)
		ergodic(node.Right)
	}
	ergodic(root)
	return res
}

func NewTreeNode(arr []int) (*TreeNode, map[int]*TreeNode) {
	l, m := len(arr), make(map[int]*TreeNode, len(arr))
	var execBuild func(start int, arr []int) *TreeNode
	execBuild = func(start int, arr []int) *TreeNode {
		if start >= l || arr[start] == -1 {
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

func TestInorderTraversal(t *testing.T) {
	assert := assert.New(t)
	root, _ := NewTreeNode([]int{1, -1, 2, -1, -1, 3})
	expected := []int{1, 3, 2}
	assert.Equal(expected, inorderTraversal(root))
}
```
