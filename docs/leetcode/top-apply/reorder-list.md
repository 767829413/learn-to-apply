# 重排链表

## 题目

给定一个单链表 L 的头节点 head ，单链表 L 表示为：

```text
L0 → L1 → … → Ln - 1 → Ln
```

请将其重新排列后变为：

```text
L0 → Ln → L1 → Ln - 1 → L2 → Ln - 2 → …
```

示例:

![1626420320-YUiulT-image.png](https://s2.loli.net/2022/09/08/sfDV8zNBJkjeZTC.png)

```text
输入：head = [1,2,3,4,5]
输出：[1,5,2,4,3]
```

---

## code

```go
// Reorder list
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reorderList(head *ListNode) {
	// 1, 2, 3, 4, 5
	// 1, 5, 2, 4, 3
	// 构建一个数组存储链表节点
	s := []*ListNode{}
	for node := head; node != nil; node = node.Next {
		s = append(s, node)
	}
	// 拿首尾节点的index
	l, r := 0, len(s)-1
	for l < r {
		s[l].Next = s[r]
		l++
		// 偶数的时候会提前结束
		if l == r {
			break
		}
		s[r].Next = s[l]
		r--
	}
	s[l].Next = nil
}

func TestReorderList(t *testing.T) {
	assert := assert.New(t)
	head, _ := NewListNode([]int{1, 2, 3, 4, 5})
	expected := []int{1, 5, 2, 4, 3}
	reorderList(head)
	assert.Equal(expected, head.GetValueArray())
}

type ListNode struct {
	Val  int
	Next *ListNode
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
