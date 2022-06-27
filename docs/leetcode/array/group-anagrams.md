# 字母异位词分组

## 题目

给你一个字符串数组，请你将 `字母异位词` 组合在一起。可以按任意顺序返回结果列表。

`字母异位词` 是由重新排列源单词的字母得到的一个新单词，所有源单词中的字母通常恰好只用一次。

示例:

```text
输入: strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
输出: [["bat"],["nat","tan"],["ate","eat","tea"]]

输入: strs = [""]
输出: [[""]]

输入: strs = ["a"]
输出: [["a"]]
```

---

## code

```go
func groupAnagrams(strs []string) [][]string {
	// sort.Slice()
	box := make([][]string, 0, len(strs))
	// 字符串字典排序后作为hash key
	// m := make(map[string][]string)
	// for _, v := range strs {
	// 	tmp := []byte(v)
	// 	sort.Slice(tmp, func(i, j int) bool { return tmp[j] > tmp[i] })
	// 	index := string(tmp)
	// 	m[index] = append(m[index], v)
	// }
	// for _, v := range m {
	// 	box = append(box, v)
	// }
	// 使用计数,字母异位词之间包含的字母必定个数相同
	c := make(map[[26]int][]string)
	for _, v := range strs {
		tmp := [26]int{}
		for _, b := range v {
			tmp[b-'a']++
		}
		c[tmp] = append(c[tmp], v)
	}
	for _, v := range c {
		box = append(box, v)
	}
	return box
}
```
