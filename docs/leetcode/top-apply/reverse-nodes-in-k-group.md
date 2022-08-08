# K 个一组翻转链表

## 题目

给定整数数组 `nums` 和整数 `k`，请返回数组中第 `k` 个最大的元素。

请注意，你需要找的是数组排序后的第 `k` 个最大的元素，而不是第 `k` 个不同的元素。

你必须设计并实现时间复杂度为 `O(n)` 的算法解决此问题。

示例:

```text
输入：head = [1,2,3,4,5], k = 2
输出：[2,1,4,3,5]

输入：head = [1,2,3,4,5], k = 3
输出：[3,2,1,4,5]
```

---

## code

```go
// Reverse nodes in k group
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseKGroup(head *ListNode, k int) *ListNode {
	// 设置哨兵,方便操作
	sentinel := &ListNode{Val: 0, Next: head}
	// 构建快慢指针节点,进行k对翻转
	pre, end := sentinel, sentinel
	// 链表翻转操作
	reverse := func(start *ListNode) *ListNode {
		var pre, cur *ListNode
		pre, cur = nil, start
		for cur != nil {
			tmp := cur.Next
			cur.Next = pre
			pre, cur = cur, tmp
		}
		return pre
	}
	for end.Next != nil {
		// 快指针节点进行k次向前
		for i := 0; i < k && end != nil; i++ {
			end = end.Next
		}
		// 如果快指针节点为nil,那么就是看无法被链表整除,剩下的不用翻转
		if end == nil {
			break
		}
		// 构建临时变量记录快,慢指针节点的下一位
		start, next := pre.Next, end.Next
		// 这里讲快指针节点下一位置空是为了方便翻转快慢指针之间的节点
		end.Next = nil
		// 执行翻转操作
		pre.Next = reverse(start)
		// 翻转后更新位置
		start.Next, pre = next, start
		end = pre
	}

	return sentinel.Next
}
```
