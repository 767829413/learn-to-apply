# 有效的括号

## 题目

给定一个只包括 `'('，')'，'{'，'}'，'['，']'` 的字符串 `s` ，判断字符串是否有效。

有效字符串需满足：

左括号必须用相同类型的右括号闭合。
左括号必须以正确的顺序闭合。

示例:

```text
输入：s = "()"
输出：true

输入：s = "([)]"
输出：false

输入：s = "{[]}"
输出：true
```

---

## code

```go
// Valid parentheses
func isValid(s string) bool {
	if s == "" {
		return false
	}
	sArr, stack, record := []byte(s), []byte{}, map[byte]byte{
		// 构建正确的括号映射组
		40: 41, 91: 93, 123: 125,
	}
	for k := range sArr {
		// 栈里的数据作为前驱进行比较
		if len(stack) > 0 && record[stack[len(stack)-1]] == sArr[k] {
			stack = stack[:len(stack)-1]
			continue
		}
		stack = append(stack, sArr[k])
	}
	return len(stack) == 0
}
```
