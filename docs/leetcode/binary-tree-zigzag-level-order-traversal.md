# 二叉树的层序遍历

## 题目

给你二叉树的根节点 root ，返回其节点值的 锯齿形层序遍历 。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

示例:

![tree1 _1_.jpg](https://s2.loli.net/2022/08/22/hGBkc7NanEH5dJe.jpg)

```text
输入：root = [3,9,20,null,null,15,7]
输出：[[3],[20,9],[15,7]]

输入：root = [1]
输出：[[1]]

输入：root = []
输出：[]
```

---

## code

```go
// Binary tree zigzag level order traversal
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func zigzagLevelOrder(root *TreeNode) [][]int {
	if root == nil {
		return [][]int{}
	}
	// 使用广度优先算法按层遍历
	// 根据遍历的层数奇偶来取值
	bfs := func(queue []*TreeNode) [][]int {
		var node *TreeNode
		res := [][]int{}
		start := 0
		// 使用队列来遍历节点数据
		for len(queue) > 0 {
			l := len(queue)
			tmpVal := []int{}
			// 取值
			if start%2 != 0 {
				for i := 0; i < l; i++ {
					tmpVal = append(tmpVal, queue[i].Val)
				}
			} else {
				for i := l - 1; i >= 0; i-- {
					tmpVal = append(tmpVal, queue[i].Val)
				}
			}

			// 处理下一次循环的队列
			for i := 0; i < l; i++ {
				node = queue[0]
				queue = queue[1:]
				if node.Right != nil {
					queue = append(queue, node.Right)
				}
				if node.Left != nil {
					queue = append(queue, node.Left)
				}
			}
			res = append(res, tmpVal)
			start++
		}
		return res
	}
	return bfs([]*TreeNode{root})
}
```
