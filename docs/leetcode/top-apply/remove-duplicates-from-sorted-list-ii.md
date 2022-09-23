# 删除排序链表中的重复元素 II

## 题目

给定一个已排序的链表的头 head ， 删除原始链表中所有重复数字的节点，只留下不同的数字 。返回 已排序的链表 。

示例:

![linkedlist1.jpg](https://s2.loli.net/2022/09/23/vpm42E6kMyNwCSc.jpg)

```text
输入：head = [1,2,3,3,4,4,5]
输出：[1,2,5]
```

---

## code

```go
// Remove duplicates from sorted list ii
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func deleteDuplicates(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return head
	}
	dumy := &ListNode{Next: head}
	pre, cur, next := dumy, dumy.Next, dumy.Next.Next
	for next != nil {
		if cur.Val == next.Val {
			if next.Next == nil || next.Next.Val != cur.Val {
				pre.Next = next.Next
				if pre.Next != nil {
					cur = pre.Next
					next = pre.Next.Next
					continue
				} else {
					break
				}
			}
			next = next.Next
			continue
		} else {
			pre, cur, next = cur, next, next.Next
		}
	}
	return dumy.Next
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

func TestDeleteDuplicates(t *testing.T) {
	assert := assert.New(t)
	head, _ := NewListNode([]int{1, 2, 3, 3, 4, 4, 5})
	expected := []int{1, 2, 5}
	assert.Equal(expected, deleteDuplicates(head).GetValueArray())
}
```
