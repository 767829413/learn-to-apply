# 文本左右对齐

## 题目

给定一个单词数组 `words` 和一个长度 `maxWidth` ，重新排版单词，使其成为每行恰好有 `maxWidth` 个字符，且左右两端对齐的文本。

你应该使用 “贪心算法” 来放置给定的单词；也就是说，尽可能多地往每行中放置单词。必要时可用空格 ' ' 填充，使得每行恰好有 `maxWidth` 个字符。

要求尽可能均匀分配单词间的空格数量。如果某一行单词间的空格不能均匀分配，则左侧放置的空格数要多于右侧的空格数。

文本的最后一行应为左对齐，且单词之间不插入额外的空格。

注意:

单词是指由非空格字符组成的字符序列。
每个单词的长度大于 `0`，小于等于 `maxWidth`。
输入单词数组 `words` 至少包含一个单词。

示例:

```text
输入: words = ["This", "is", "an", "example", "of", "text", "justification."], maxWidth = 16
输出:
[
   "This    is    an",
   "example  of text",
   "justification.  "
]

输入:words = ["What","must","be","acknowledgment","shall","be"], maxWidth = 16
输出:
[
  "What   must   be",
  "acknowledgment  ",
  "shall be        "
]
解释: 注意最后一行的格式应为 "shall be    " 而不是 "shall     be",
     因为最后一行应为左对齐，而不是左右两端对齐。       
     第二行同样为左对齐，这是因为这行只包含一个单词。

输入:words = ["Science","is","what","we","understand","well","enough","to","explain","to","a","computer.","Art","is","everything","else","we","do"]，maxWidth = 20
输出:
[
  "Science  is  what we",
  "understand      well",
  "enough to explain to",
  "a  computer.  Art is",
  "everything  else  we",
  "do                  "
]
```

---

## code

```go
// Text justification
func fullJustify(words []string, maxWidth int) []string {
	l := len(words)
	box := []string{}
	var builder strings.Builder
	if l == 0 {
		for i := 0; i < maxWidth; i++ {
			builder.WriteString(" ")
		}
		return []string{builder.String()}
	}

	getP := func(nsn, indexN int) int {
		var p, p1, p2 int
		p1 = nsn % indexN
		p2 = nsn / indexN
		if p1 == 0 {
			p = p2
		} else {
			p = p2 + 1
		}
		return p
	}

	setSpace := func(nsn int, res []string) string {
		end := len(res) - 1
		indexN := end
		p := getP(nsn, indexN)
		newR := []string{}
		for k := range res {
			newR = append(newR, res[k])
			if k == end {
				break
			}
			if k == end-1 {
				for i := 0; i < nsn; i++ {
					newR = append(newR, " ")
				}
			} else {
				for i := 0; i < p; i++ {
					newR = append(newR, " ")
				}
				indexN--
				nsn -= p
				p = getP(nsn, indexN)
			}

		}
		return strings.Join(newR, "")
	}
	setEndSpace := func(nsn int, res []string) string {
		end := len(res) - 1
		indexN := end
		if indexN == 0 {
			for i := 0; i < nsn; i++ {
				res = append(res, " ")
			}
			return strings.Join(res, "")
		} else {
			tmp := []string{}
			for k := range res {
				tmp = append(tmp, res[k])
				if k == end {
					for i := 0; i < nsn; i++ {
						tmp = append(tmp, " ")
					}
					break
				} else {
					tmp = append(tmp, " ")
					nsn--
				}
			}
			return strings.Join(tmp, "")
		}
	}
	type tmp struct {
		len     int
		realLen int
		str     []string
	}
	t := &tmp{
		len: 0,
		str: []string{},
	}
	for i := 0; i < l; i++ {
		sn := maxWidth - t.realLen
		flag := maxWidth - t.len - len(words[i])

		if flag <= 0 {
			if flag == 0 {
				t.str = append(t.str, words[i])
				t.realLen = t.realLen + len(words[i])
				t.len = t.len + len(words[i]) + 1
				sn = maxWidth - t.realLen
			}
			needStr := ""
			if len(t.str) == 1 {
				needStr = setEndSpace(sn, t.str)
			} else {
				needStr = setSpace(sn, t.str)
			}
			box = append(box, needStr)
			t = &tmp{
				len:     0,
				realLen: 0,
				str:     []string{},
			}
		}

		if i == l-1 && flag != 0 {
			t.str = append(t.str, words[i])
			t.realLen = t.realLen + len(words[i])
			t.len = t.len + len(words[i]) + 1
			sn = maxWidth - t.realLen
			needStr := setEndSpace(sn, t.str)
			box = append(box, needStr)
			return box
		}

		if flag != 0 {
			t.str = append(t.str, words[i])
			t.realLen = t.realLen + len(words[i])
			t.len = t.len + len(words[i]) + 1
		}

	}
	return box
}
```
