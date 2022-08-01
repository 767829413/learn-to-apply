# 子集 II

## 题目

给定两个整数数组 preorder 和 inorder ，其中 preorder 是二叉树的先序遍历， inorder 是同一棵树的中序遍历，请构造二叉树并返回其根节点。

示例:

![histogram.jpg](https://s2.loli.net/2022/08/01/9kvKDPxsYWjCdim.jpg)

```text
输入: preorder = [3,9,20,15,7], inorder = [9,3,15,20,7]
输出: [3,9,20,null,null,15,7]
```

---

## code

```go
// Construct binary tree from preorder and inorder traversal
func buildTree(preorder []int, inorder []int) *TreeNode {
	pL := len(preorder)
	iL := len(inorder)
	if pL != iL {
		panic("Illegal input")
	}
	m := make(map[int]int, iL)
	for k, v := range inorder {
		m[v] = k
	}
	var execBuildTree func(pl, pr, il, ir int) *TreeNode
	execBuildTree = func(pl, pr, il, ir int) *TreeNode {
		// 终止条件
		if pl > pr || il > ir {
			return nil
		}
		// fmt.Println(pl, pr, il, ir)
		// fmt.Println(preorder[pl])
		// fmt.Println(m[preorder[pl]])
		// 先序遍历的顺序是根节点，左子树，右子树
		// 中序遍历的顺序是左子树，根节点，右子树
		// 前序首位就是当前层根节点
		// 只需要根据先序遍历得到根节点，然后在中序遍历中找到根节点的位置，它的左边就是左子树的节点，右边就是右子树的节点
		rootVal := preorder[pl]
		rootInIndex := m[rootVal]
		root := &TreeNode{
			Val:   rootVal,
			Left:  execBuildTree(pl+1, pl+rootInIndex-il, il, rootInIndex-1),
			Right: execBuildTree(pl+rootInIndex-il+1, pr, rootInIndex+1, ir),
		}
		return root
	}
	return execBuildTree(0, pL-1, 0, iL-1)
}

type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}
```
