# 排序链表

## 题目

给你链表的头结点 head ，请将其按 升序 排列并返回 排序后的链表 。

示例:

![sort_list_1.jpg](https://s2.loli.net/2022/09/22/S39uONsVozl5UpM.jpg)

```text
输入：head = [4,2,1,3]
输出：[1,2,3,4]
```

---

## code

```go
// Sort list
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func sortList(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return head
	}
	var normalFunc = func(head *ListNode) *ListNode {
		m := []*ListNode{}
		for i := head; i != nil; i = i.Next {
			m = append(m, i)
		}
		sort.Slice(m, func(i, j int) bool {
			return m[i].Val < m[j].Val
		})
		l := len(m) - 1
		for i := 0; i < l; i++ {
			m[i].Next = m[i+1]

		}
		m[l].Next = nil
		return m[0]
	}
	return normalFunc(head)
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

func TestSortList(t *testing.T) {
	assert := assert.New(t)
	head, _ := NewListNode([]int{4, 2, 1, 3})
	expected := []int{1, 2, 3, 4}
	assert.Equal(expected, sortList(head).GetValueArray())
}
```
