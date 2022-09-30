# 最长公共子序列

## 题目

给定两个字符串 `text1` 和 `text2`，返回这两个字符串的最长 `公共子序列` 的长度。如果不存在 `公共子序列` ，返回 0 。

一个字符串的 `子序列` 是指这样一个新的字符串：它是由原字符串在不改变字符的相对顺序的情况下删除某些字符（也可以不删除任何字符）后组成的新字符串。

    例如，"ace" 是 "abcde" 的子序列，但 "aec" 不是 "abcde" 的子序列。

两个字符串的 `公共子序列` 是这两个字符串所共同拥有的子序列。

示例:

```text
输入：text1 = "abcde", text2 = "ace" 
输出：3  
解释：最长公共子序列是 "ace" ，它的长度为 3 。

输入：text1 = "abc", text2 = "abc"
输出：3
解释：最长公共子序列是 "abc" ，它的长度为 3 。

输入：text1 = "abc", text2 = "def"
输出：0
解释：两个字符串没有公共子序列，返回 0 。
```

---

## code

```go
// Longest common subsequence
func longestCommonSubsequence(text1 string, text2 string) int {
	l1, l2 := len(text1), len(text2)

	if l1 == 0 || l2 == 0 {
		return 0
	}
	// 动态规划求解,首先定义状态数组 dp[i][j]
	// dp[i][j] 表示 text1[0,i-1] 与 text2[0,j-1] 之间的最长公共子序列数
	// dp[0][0] 表示空字符串,所以要考虑增加空字符串的可能性
	dp := make([][]int, l1+1)
	for k := range dp {
		dp[k] = make([]int, l2+1)
	}
	// 构建状态转移方程
	// 如果 text1[i-1] == text2[j-1],则表示末尾相同,那么 dp[i][j] = dp[i-1][j-1] + 1
	// 反之, dp[i][j] = max(dp[i][j-1], dp[i-1][j])
	b1, b2, max := []byte(text1), []byte(text2), func(a, b int) int {
		if a > b {
			return a
		} else {
			return b
		}
	}
	for i := 1; i <= l1; i++ {
		for j := 1; j <= l2; j++ {
			if b1[i-1] == b2[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[l1][l2]
}
```
