# 合并K个升序链表

## 题目

给你一个链表数组，每个链表都已经按升序排列。

请你将所有链表合并到一个升序链表中，返回合并后的链表。

示例:

```text
输入：lists = [[1,4,5],[1,3,4],[2,6]]
输出：[1,1,2,3,4,4,5,6]
解释：链表数组如下：
[
  1->4->5,
  1->3->4,
  2->6
]
将它们合并到一个有序链表中得到。
1->1->2->3->4->4->5->6

输入：lists = []
输出：[]
```

---

## code

```go
// Merge k sorted lists
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeKLists(lists []*ListNode) *ListNode {
	l := len(lists)
	if l == 0 {
		return nil
	}
	if l == 1 {
		return lists[0]
	}
	// 合并两个有序数组
	mergeTwoLists := func(list1 *ListNode, list2 *ListNode) *ListNode {
		head := &ListNode{}
		start := head
		for list1 != nil && list2 != nil {
			if list1.Val > list2.Val {
				start.Next = list2
				list2 = list2.Next
			} else {
				start.Next = list1
				list1 = list1.Next
			}
			start = start.Next
		}
		if list1 != nil {
			start.Next = list1
		} else {
			start.Next = list2
		}
		return head.Next
	}

	// 采用分治法
	var merge func(lists []*ListNode, left, right int) *ListNode
	merge = func(lists []*ListNode, left, right int) *ListNode {
		// 递归结束条件就是左右相等 || 左边大于右边
		if left == right {
			return lists[left]
		}
		if left > right {
			return nil
		}
		mid := (left + right) >> 1
		return mergeTwoLists(merge(lists, left, mid), merge(lists, mid+1, right))
	}

	return merge(lists, 0, l-1)
}

func TestMergeKLists(t *testing.T) {
	assert := assert.New(t)
	list1, _ := NewListNode([]int{1, 4, 5})
	list2, _ := NewListNode([]int{1, 3, 4})
	list3, _ := NewListNode([]int{2, 6})
	lists := []*ListNode{list1, list2, list3}
	expected := []int{1, 1, 2, 3, 4, 4, 5, 6}
	assert.Equal(expected, mergeKLists(lists).GetValueArray())
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
