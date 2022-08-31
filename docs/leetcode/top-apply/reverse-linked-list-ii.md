# 反转链表 II

## 题目

给你单链表的头指针 head 和两个整数 left 和 right ，其中 left <= right 。请你反转从位置 left 到位置 right 的链表节点，返回 反转后的链表 。

示例:

![rev2ex2.jpg](https://s2.loli.net/2022/08/31/ZzutxRBbMcGeJAy.jpg)

```text
输入：head = [1,2,3,4,5], left = 2, right = 4
输出：[1,4,3,2,5]
```

---

## code

```go
// Reverse linked list ii
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseBetween(head *ListNode, left int, right int) *ListNode {
	if left == right {
		return head
	}
	var pre, start, end, next *ListNode
	c, s := 1, head
	for s != nil {
		if c+1 == left {
			pre, start = s, s.Next
		}
		if c == left {
			start = s
		}
		if c == right {
			end, next = s, s.Next
		}
		s = s.Next
		c++
	}
	if start == nil || end == nil {
		return head
	}
	end.Next = nil
	reverseList := func(s *ListNode) *ListNode {
		var pre *ListNode
		pre, cur := nil, s
		for cur != nil {
			tmp := cur.Next
			cur.Next = pre
			cur, pre = tmp, cur
		}
		return pre
	}
	if pre != nil {
		pre.Next = reverseList(start)
	} else {
		reverseList(start)
		head = end
	}
	start.Next = next
	return head
}

func TestReverseBetween(t *testing.T) {
	assert := assert.New(t)
	head, _ := NewListNode([]int{3, 5})
	left, right := 1, 2
	expected := []int{5, 3}
	assert.Equal(expected, reverseBetween(head, left, right).GetValueArray())
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
```
