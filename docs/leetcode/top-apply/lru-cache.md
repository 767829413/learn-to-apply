# LRU 缓存

## 题目

请你设计并实现一个满足 `LRU (最近最少使用) 缓存` 约束的数据结构。
实现 `LRUCache` 类：
`LRUCache(int capacity)` 以 `正整数` 作为容量 `capacity` 初始化 `LRU 缓存`
`int get(int key)` 如果关键字 `key` 存在于缓存中，则返回关键字的值，否则返回 `-1` 。
`void put(int key, int value)` 如果关键字 `key` 已经存在，则变更其数据值 `value` ；如果不存在，则向缓存中插入该组 `key-value` 。如果插入操作导致关键字数量超过 `capacity` ，则应该 `逐出` 最久未使用的关键字。
函数 `get` 和 `put` 必须以 `O(1)` 的平均时间复杂度运行。

示例:

```text
输入
["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
[[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]
输出
[null, null, null, 1, null, -1, null, -1, 3, 4]

解释
LRUCache lRUCache = new LRUCache(2);
lRUCache.put(1, 1); // 缓存是 {1=1}
lRUCache.put(2, 2); // 缓存是 {1=1, 2=2}
lRUCache.get(1);    // 返回 1
lRUCache.put(3, 3); // 该操作会使得关键字 2 作废，缓存是 {1=1, 3=3}
lRUCache.get(2);    // 返回 -1 (未找到)
lRUCache.put(4, 4); // 该操作会使得关键字 1 作废，缓存是 {4=4, 3=3}
lRUCache.get(1);    // 返回 -1 (未找到)
lRUCache.get(3);    // 返回 3
lRUCache.get(4);    // 返回 4
```

---

## code

```go
/**
 * Your LRUCache object will be instantiated and called as such:
 * obj := Constructor(capacity);
 * param_1 := obj.Get(key);
 * obj.Put(key,value);
 */
type LRUCache struct {
	capacity int
	length   int
	data     map[int]*InteractiveList
	head     *InteractiveList // 哨兵节点
	tail     *InteractiveList // 哨兵节点
}

func Constructor(capacity int) LRUCache {
	if capacity == 0 {
		panic("Wrong input")
	}

	return LRUCache{
		capacity: capacity,
		data:     make(map[int]*InteractiveList, capacity),
		head:     &InteractiveList{},
		tail:     &InteractiveList{},
	}
}

func (this *LRUCache) Get(key int) int {
	if this.length == 0 {
		return -1
	}
	if v, ok := this.data[key]; ok {
		this.replaceNew(v)
		return v.Val
	}
	return -1
}

func (this *LRUCache) Put(key int, value int) {
	if v, ok := this.data[key]; ok {
		v.Val = value
		this.replaceNew(v)
		return
	}
	if this.length == this.capacity {
		this.refresh()
	}
	node := &InteractiveList{Key: key, Val: value}
	this.data[key] = node
	if this.length == 0 {
		node.Pre, node.Next = this.head, this.tail
		this.head.Next = node
		this.tail.Pre = node
		this.length++
		return
	}
	preTmp := this.tail.Pre
	node.Pre = preTmp
	node.Next = this.tail
	preTmp.Next = node
	this.tail.Pre = node
	this.length++
}

// 存储空间满了,开始移除最久未使用的
func (this *LRUCache) refresh() {
	delete(this.data, this.head.Next.Key)
	nextTmp := this.head.Next.Next
	if nextTmp != nil {
		nextTmp.Pre = this.head
	}
	this.head.Next = nextTmp
	this.length--
}

// 刷新数据,保持使用的在前面
func (this *LRUCache) replaceNew(v *InteractiveList) {
	v.Pre.Next = v.Next
	v.Next.Pre = v.Pre
	this.tail.Pre.Next = v
	v.Pre = this.tail.Pre
	v.Next = this.tail
	this.tail.Pre = v
}

type InteractiveList struct {
	Key  int
	Val  int
	Next *InteractiveList
	Pre  *InteractiveList
}
```
