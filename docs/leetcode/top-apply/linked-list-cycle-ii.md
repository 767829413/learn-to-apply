# 环形链表 II

## 题目

给定一个链表的头节点  head ，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。

如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。 为了表示给定链表中的环，评测系统内部使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。如果 pos 是 -1，则在该链表中没有环。注意：pos 不作为参数进行传递，仅仅是为了标识链表的实际情况。

不允许修改 链表。

示例:

![circularlinkedlist.png](https://s2.loli.net/2022/09/01/n6lRzx7dHv5fPJe.png)

```text
输入：head = [3,2,0,-4], pos = 1
输出：返回索引为 1 的链表节点
解释：链表中有一个环，其尾部连接到第二个节点。
```

---

## code

```go
// Linked list cycle ii
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func detectCycle(head *ListNode) *ListNode {
	if head == nil {
		return nil
	}
	s, f := head, head
	// 定义环形链表环入口之前的节点数 a,环节点数 b
	// 找两个指针重合位置,此时
	// f(快指针,走两步)走了的节点数设为 x ,s(慢指针,走一步)走了的节点数设为 y
	// f比s多走一倍的节点数 x = 2y,因为环状链表, x = y + n*b (f比s多走了n圈环才能相遇的)
	// y = nb, x = 2*nb,重合时 s 走了 n 个环长, f 走了 2n 个环长
	// 从链表头走到环形入口的节点数是: a + n*b (环形入口后都是环,所以绕 n 都能回入口)
	// 此时 s,f 相遇,那么 f 再走 a 个节点则能到达入口
	// 利用一个新指针 h ,从链表头开始走 a 个节点则: h 与 f 相遇则为入口节点
	for {
		// 标识无环
		if f == nil || f.Next == nil {
			return nil
		}
		f = f.Next.Next
		s = s.Next
		if f == s {
			break
		}
	}
	f = head
	for s != f {
		f, s = f.Next, s.Next
	}
	return f
}

func TestDetectCycle(t *testing.T) {
	assert := assert.New(t)
	head, m := NewListNode([]int{3, 2, 0, -4})
	pos := 1
	m[3].Next = m[pos]
	expected := m[pos]
	assert.Equal(expected, detectCycle(head))
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
```
