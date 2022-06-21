# 缺失的第一个正数

## 题目

给你一个未排序的整数数组 **nums** ，请你找出其中没有出现的最小的正整数。

请你实现时间复杂度为 **O(n)** 并且只使用常数级别额外空间的解决方案。

示例:

```text
输入：nums = [1,2,0]
输出：3

输入：nums = [3,4,-1,1]
输出：2

输入：nums = [7,8,9,11,12]
输出：1
```

---

## code

```go
func firstMissingPositive(nums []int) int {
	l := len(nums)
	// 建立hash表,存储所有nums 大于0的值
	m := make(map[int]int, l)
	for _, v := range nums {
		if v <= 0 {
			continue
		}
		m[v] = 1
	}
	// 从1到l进行遍历,因为找的是最小的未出现的数,转换为hash表里未出现的最小数即可
	// 为啥终止条件是l呢? 如果是 [1~n](顺序+1递增)那么nums长为n,最终最小值也就为n+1了
	// 如果是 [x~n](非顺序+1递增) 那么 1到n遍历最小的值肯定在1到l之间
	for i := 1; i <= l; i++ {
		if _, ok := m[i]; !ok {
			return i
		}
	}
	return l + 1
}
```
