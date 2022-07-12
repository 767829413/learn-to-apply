# 加一

## 题目

给定一个由 `整数` 组成的 `非空` 数组所表示的非负整数，在该数的基础上加一。

最高位数字存放在数组的首位， 数组中每个元素只存储 `单个数字` 。

你可以假设除了整数 `0` 之外，这个整数不会以零开头。

示例:

```text
输入：digits = [1,2,3]
输出：[1,2,4]
解释：输入数组表示数字 123。

输入：nums = [9]
输出：[1,0]
```

---

## code

```go
// Plus one
func plusOne(digits []int) []int {
	l := len(digits)
	// 取最后一位数开始
	for i := l - 1; i >= 0; i-- {
		digits[i]++
		// 取余
		digits[i] %= 10
		// 取余不为0则返回
		if digits[i] != 0 {
			return digits
		}
	}
	// 到这就是全体进一位,扩充切片
	return append([]int{1}, digits...)
}
```
