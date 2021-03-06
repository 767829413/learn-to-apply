
# 了解下字符串匹配,初尝BM算法-2

    由BM算法(好后缀,坏字符原则)到KMP(好前缀原则)算法,主串不动,模式串多滑动

## **演示Code**

```go
package string

func KnuthMorrisPrattMatch(a, b []rune, n, m int) int {
   next := getNext(b, m)
   j := 0
   for i := 0; i < n; i++ {
      for j > 0 && j < m && a[i] != b[j] {
         j = next[j-1] + 1
      }
      if a[i] == b[j] {
         j++
      }
      if j == m {
         return i - m + 1
      }
   }
   return -1
}

func getNext(b []rune, m int) []int {
   next := make([]int, m, m)
   next[0] = -1
   index := -1
   for i := 1; i < m; i++ {
      for index != -1 && b[index+1] != b[i] {
         index = next[index]
      }
      if b[index+1] == b[i] {
         index++
      }
      next[i] = index
   }
   return next
}
```

## **测试Code**

```go
package string

import "testing"

func TestKnuthMorrisPrattMatch(t *testing.T) {
   str1 := `毛选第五卷包括毛泽东在中华人民共和国建国后的文章共70篇，截至1957年。1977年4月15日由人民出版社正式出版发行，定价1.25元人民币。
本卷1967年开始筹备，原负责筹备工作的是中央文革小组理论组，以周恩来为名义上的负责人，不久即停止。1969年后，在康生的领导下再次开始准备工作，但不久后因未获得毛泽东的认可，工作又一次停止。[5]
毛泽东逝世后，华国锋决定继续出版毛选第五卷，在击溃四人帮后，于1976年10月8日再度开始筹备工作，由汪东兴为名义负责人，李鑫、吴泠西、胡绳、熊复实际负责编辑工作。
第五卷出版后，华国锋仍保留了毛泽东选集编辑委员会，准备筹备出版毛选第六卷（未出版），但随着邓小平的掌握政权，中共中央不再坚持对毛泽东的个人崇拜。1980年，毛泽东选集编辑委员会改组为中共中央文献研究室，此后未再出版毛选。[6]已发售的毛选第五卷，也在1982年后停售，目前只能在一些二手书刊交易平台购得。
为了最终弥补“毛选只有部分毛泽东文章，尤其没有建国后文章”的缺憾，中共中央在1990年代又组织编印了《毛泽东文集》、《毛泽东早期文稿》（内部发行）、《建国以来毛泽东文稿》（内部发行）等著作集。`
   str2 := "《毛泽东"
   str1Rune := []rune(str1)
   str2Rune := []rune(str2)
   res := KnuthMorrisPrattMatch(str1Rune, str2Rune, len(str1Rune), len(str2Rune))
   t.Log(res) //458
}
```
