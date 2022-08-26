# 二叉树的最近公共祖先

## 题目

给你一个字符串 s，找到 s 中最长的回文子串。

示例:

```text
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。

输入：s = "cbbd"
输出："bb"
```

---

## code

```go
// Longest palindromic substring
func longestPalindrome(s string) string {
	sArr := []byte(s)
	l := len(sArr)
	if l < 2 {
		return s
	}
	res, maxL := [2]int{}, math.MinInt
	// 中心位置寻找
	findFunc := func(sArr []byte, left, right int) [2]int {
		for left >= 0 && right < l {
			if sArr[left] == sArr[right] {
				left--
				right++
			} else {
				break
			}
		}
		return [2]int{left + 1, right - left - 1}
	}
	// 遍历数组
	for k := range sArr {
		odd := findFunc(sArr, k, k)
		even := findFunc(sArr, k, k+1)
		if odd[1] > even[1] {
			even = odd
		}
		if maxL < even[1] {
			res = even
			maxL = even[1]
		}
	}

	return s[res[0] : res[0]+res[1]]
}
```
