# 移除元素

## 题目

给你单链表的头节点 head ，请你反转链表，并返回反转后的链表。

示例:

![rev1ex1.jpg](https://s2.loli.net/2022/08/02/4HqfwPghoCV8uYF.jpg)

```text
输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]
```

---

## code

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
	var pre, cur *ListNode
	pre, cur = nil, head
	for cur != nil {
		tmp := cur.Next
		cur.Next = pre
		pre, cur = cur, tmp
	}
	return pre
}

type ListNode struct {
	Val  int
	Next *ListNode
}

func NewListNode(Values []int) *ListNode {
	l := len(Values)
	if l == 0 {
		panic("Wrong input")
	}
	var recurrence func(start int) *ListNode
	recurrence = func(start int) *ListNode {
		if start == l {
			return nil
		}
		node := &ListNode{Val: Values[start]}
		node.Next = recurrence(start + 1)
		return node
	}
	return recurrence(0)
}
```
