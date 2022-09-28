# 下一个排列

## 题目

给你两个 `非空` 的链表，表示两个非负的整数。它们每位数字都是按照 `逆序` 的方式存储的，并且每个节点只能存储 `一位` 数字。

请你将两个数相加，并以相同形式返回一个表示和的链表。

你可以假设除了数字 `0` 之外，这两个数都不会以 `0` 开头。

示例:

![addtwonumber1.jpg](https://s2.loli.net/2022/09/28/Rc4HGBgNewioJ9U.jpg)

```text
输入：l1 = [2,4,3], l2 = [5,6,4]
输出：[7,0,8]
解释：342 + 465 = 807.
```

---

## code

```go
// Add two numbers
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
	head, progression := &ListNode{}, 0
	s := head
	var s1, s2 *ListNode
	setFunc := func(a, b, c int, s *ListNode) (*ListNode, int) {
		tmp := a + b + c
		c = tmp / 10
		s.Next = &ListNode{Val: tmp % 10}
		s = s.Next
		return s, c
	}
	for s1, s2 = l1, l2; s1 != nil && s2 != nil; s1, s2 = s1.Next, s2.Next {
		s, progression = setFunc(s1.Val, s2.Val, progression, s)
	}

	endFunc := func(end *ListNode) {
		for ; end != nil; end = end.Next {
			s, progression = setFunc(0, end.Val, progression, s)
		}
		if progression == 1 {
			s.Next = &ListNode{Val: 1}
		}
	}

	if s1 == nil {
		endFunc(s2)
	} else {
		endFunc(s1)
	}
	return head.Next
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

func TestAddTwoNumbers(t *testing.T) {
	assert := assert.New(t)
	l1, _ := NewListNode([]int{9,9,9,9,9,9,9})
	l2, _ := NewListNode([]int{9,9,9,9})
	expected := []int{8,9,9,9,0,0,0,1}
	assert.Equal(expected, addTwoNumbers(l1, l2).GetValueArray())
}
```
