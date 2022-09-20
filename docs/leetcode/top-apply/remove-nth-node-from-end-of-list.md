# 删除链表的倒数第 N 个结点

## 题目

给你一个链表，删除链表的倒数第 n 个结点，并且返回链表的头结点。

示例:

![remove_ex1.jpg](https://s2.loli.net/2022/09/20/EpHm7XlahwzGMy4.jpg)

```text
输入：head = [1,2,3,4,5], n = 2
输出：[1,2,3,5]
```

---

## code

```go
// Remove nth node from end of list
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func removeNthFromEnd(head *ListNode, n int) *ListNode {
	m := make([]*ListNode, 0, 500)
	for s := head; s != nil; s = s.Next {
		m = append(m, s)
	}
	l := len(m)
	index := l - n
	if l < 2 {
		return nil
	}
	switch index {
	case l-1:
		m[index-1].Next = nil
	case 0:
		head = head.Next
	default:
		m[index-1].Next = m[index].Next
	}
	return head
}

func NewListNode(Values []int) (*ListNode, map[int]*ListNode) {
	l := len(Values)
	if l == 0 {
		panic("Wrong input")
	}
	m := make(map[int]*ListNode, len(Values))
	var recurrence func(start int) *ListNode
	recurrence = func(start int) *ListNode {
		if start == l {
			return nil
		}
		node := &ListNode{Val: Values[start]}
		m[start] = node
		node.Next = recurrence(start + 1)
		return node
	}
	return recurrence(0), m
}

func (ln *ListNode) GetValueArray() []int {
	res := []int{}
	cur := ln
	for cur != nil {
		res = append(res, cur.Val)
		cur = cur.Next
	}
	return res
}

func TestRemoveNthFromEnd(t *testing.T) {
	assert := assert.New(t)
	head, _ := NewListNode([]int{1, 2, 3, 4, 5})
	n := 2
	expected := []int{1, 2, 3, 5}
	assert.Equal(expected, removeNthFromEnd(head, n).GetValueArray())
}
```
