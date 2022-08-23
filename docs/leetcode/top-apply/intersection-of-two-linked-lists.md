# 相交链表

## 题目

给你两个单链表的头节点 headA 和 headB ，请你找出并返回两个单链表相交的起始节点。如果两个链表不存在相交节点，返回 null 。

示例:

![160_statement.png](https://s2.loli.net/2022/08/23/eS93nm7QqoOAuZU.png)

```text
输入：headA = [a1, a2, c1, c2, c3], headB = [b1, b2, b3, c1, c2, c3] 
输出：c1
```

---

## code

```go
// Ontersection of two linked lists
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func getIntersectionNode(headA, headB *ListNode) *ListNode {
	// 使用hash表来记录 O(m+n) O(m)
	/*
		hashAuxiliary := func(headA, headB *ListNode) *ListNode {
			m := make(map[*ListNode]interface{})
			for headA != nil {
				m[headA] = nil
				headA = headA.Next
			}
			for headB != nil {
				if _, ok := m[headB]; ok {
					return headB
				}
				headB = headB.Next
			}
			return nil
		}
	*/
	// 采用跑圈原理,两个指针a b,a先跑A链再跑B链, b先跑B链再跑A链,如果二者能相遇,那就是重合的第一个节点相遇
	// a跑完 A+B链 或 b跑完B+A链则表示无重合
	a, b, flaga, flagb := headA, headB, false, false
	for a != b {
		a = a.Next
		if a == nil && !flaga {
			flaga = true
			a = headB
		}
		b = b.Next
		if b == nil && !flagb {
			flagb = true
			b = headA
		}
	}
	return a
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

func TestGetIntersectionNode(t *testing.T) {
	assert := assert.New(t)
	headA := NewListNode([]int{1, 9, 1})
	headB := NewListNode([]int{3, 5})
	var expected *ListNode
	Intersection := NewListNode([]int{2, 4, 6})
	headA.Next.Next = Intersection
	headB.Next = Intersection
	expected = Intersection
	assert.Equal(expected, getIntersectionNode(headA, headB))
	// assert.Equal(expected, getIntersectionNode(headA, headB))
}
```
