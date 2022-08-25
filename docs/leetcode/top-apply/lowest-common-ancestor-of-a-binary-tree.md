# 二叉树的最近公共祖先

## 题目

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

百度百科中最近公共祖先的定义为：“对于有根树 T 的两个节点 p、q，最近公共祖先表示为一个节点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

示例:

![binarytree.png](https://s2.loli.net/2022/08/25/9qZzb3Oyg8EH6YQ.png)

```text
输入：root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1
输出：3
解释：节点 5 和节点 1 的最近公共祖先是节点 3 。

输入：root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 4
输出：5
解释：节点 5 和节点 4 的最近公共祖先是节点 5 。因为根据定义最近公共祖先节点可以为节点本身。
```

---

## code

```go
// Lowest common ancestor of a binary tree
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	// 使用深度优先进行递归操作
	// 因为 p q 节点本身也可以是最近公共祖先
	// 从root开始中序遍历(遍历顺序无要求),可以分为以下几种情况
	// 1. 如果左右返回nil,表示该root下,p,q没有公共祖先
	// 2. 左右均不空,那root本身就是最近公共祖先
	// 3. 左不为空,右为空,左为最近公共祖先,左为空,右不为空,右为最近公共祖先
	var dfs func(node *TreeNode) *TreeNode
	dfs = func(node *TreeNode) *TreeNode {
		if node == p || node == q || node == nil {
			return node
		}
		left := dfs(node.Left)
		right := dfs(node.Right)

		switch true {
		case left == right && right == nil:
			return nil
		case left == nil && right != nil:
			return right
		case left != nil && right == nil:
			return left
		case left != nil && right != nil:
			return node
		}
		return nil
	}
	return dfs(root)
}

func TestLowestCommonAncestor(t *testing.T) {
	assert := assert.New(t)
	root, m := NewTreeNode([]int{3, 5, 1, 6, 2, 0, 8, -1, -1, 7, 4})
	expected := m[1]
	assert.Equal(expected, lowestCommonAncestor(root, m[1], m[10]))
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

```
