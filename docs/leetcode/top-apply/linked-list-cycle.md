# 环形链表

## 题目

给定一个数组 prices ，它的第 i 个元素 prices[i] 表示一支给定股票第 i 天的价格。

你只能选择 某一天 买入这只股票，并选择在 未来的某一个不同的日子 卖出该股票。设计一个算法来计算你所能获取的最大利润。

返回你可以从这笔交易中获取的最大利润。如果你不能获取任何利润，返回 0 。

示例:

![circularlinkedlist.png](https://s2.loli.net/2022/08/18/TVA1i8mKpY2xehI.png)

```text
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```

---

## code

```go
// Linked list cycle
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func hasCycle(head *ListNode) bool {
	if head == nil || head.Next == nil {
		return false
	}
	// 通过快慢指针,两个指针只要相遇,必有环
	// 注意: 快指针的快是每次走的快
	slow, fast := head, head.Next
	for fast != nil {
		if slow == fast {
			return true
		}
		if fast.Next == nil {
			return false
		}
		fast = fast.Next.Next
		slow = slow.Next
	}
	return false
}
```
