# 字符串相加

## 题目

给定两个字符串形式的非负整数 num1 和num2 ，计算它们的和并同样以字符串形式返回。

你不能使用任何內建的用于处理大整数的库（比如 BigInteger）， 也不能直接将输入的字符串转换为整数形式。

示例:

```text
输入：num1 = "11", num2 = "123"
输出："134"
```

---

## code

```go
// Add strings
func addStrings(num1 string, num2 string) string {
	res, i, j, up := "", len(num1)-1, len(num2)-1, 0
	// 利用两个字符串数字从后相加
	// 利用十进制进位来处理 十进制数 xyz, z / 10 标识是否进位 z % 10 标识当前位置 z 的值
	for i >= 0 || j >= 0 {
		var n1, n2 = 0, 0
		if i >= 0 {
			n1, _ = strconv.Atoi(string(num1[i]))
		}
		if j >= 0 {
			n2, _ = strconv.Atoi(string(num2[j]))
		}
		tmp := n1 + n2 + up
		up = tmp / 10
		res = strconv.Itoa(tmp%10) + res
		i--
		j--
	}
	if up != 0 {
		res = "1" + res
	}
	return res
}
```
