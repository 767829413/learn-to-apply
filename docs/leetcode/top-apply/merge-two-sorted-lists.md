# 合并两个有序链表

## 题目

将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

示例:

![merge_ex1.jpg](https://s2.loli.net/2022/08/12/cRqY9yv8Vxa7Ni2.jpg)

```text
输入：l1 = [1,2,4], l2 = [1,3,4]
输出：[1,1,2,3,4,4]

输入：l1 = [], l2 = []
输出：[]

输入：l1 = [], l2 = [0]
输出：[0]
```

---

## code

```go
// Merge two sorted lists
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
	// 哨兵节点
	head := &ListNode{}
	start := head
	for list1 != nil && list2 != nil {
		if list1.Val < list2.Val {
			start.Next = list1
			list1 = list1.Next
		} else {
			start.Next = list2
			list2 = list2.Next
		}
		start = start.Next
	}
	t := list1
	if list1 == nil {
		t = list2
	}
	for t != nil {
		start.Next = t
		t = t.Next
		start = start.Next
	}
	return head.Next
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
