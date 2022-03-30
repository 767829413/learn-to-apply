
# 了解下字符串匹配,初尝BM算法-1

## **演示Code**
```
package string

import (
   "math"
)

func BoyerMooreMatch(a, b []rune, n, m int) int {
   //记录模式串中每个字符最后出现的位置
   hash := generateHash(b, m)
   //后缀匹配预处理
   suffix, prefix := generatePre(b, m)
   //i表示主串与模式串对齐的第一个字符
   i := 0
   num := n - m
   for i <= num {
      j := m - 1
      for ; j >= 0; j-- {
         if a[i+j] != b[j] {
            break
         }
      }
      if j < 0 {
         return i
      }
      //这里等同于将模式串往后滑动j-hash[a[i+j]]位
      x := j - hash[a[i+j]]
      y := 0
      if j < m-1 {
         y = moveByPre(j, m, suffix, prefix)
      }
      ll := int(math.Max(float64(x), float64(y)))
      if ll == 0 {
         i++
      } else {
         i = i + ll
      }

   }
   return -1
}

//构建坏字符规则的hash表,记录模式串中字符位置
func generateHash(b []rune, m int) map[rune]int {
   hash := make(map[rune]int, m)
   for k, v := range b {
      hash[v] = k
   }
   return hash
}

//预处理好后缀规则,返回移动距离
func generatePre(b []rune, m int) (suffix []int, prefix []bool) {
   suffix = make([]int, m, m)
   prefix = make([]bool, m, m)
   for i := 0; i < m; i++ {
      suffix[i] = -1
      prefix[i] = false
   }
   for i := 0; i < m-1; i++ {
      j := i
      l := 0
      for j >= 0 && b[j] == b[m-1-l] {
         j--
         l++
         suffix[l] = j + 1
      }
      if j == -1 {
         prefix[l] = true
      }
   }
   return
}

func moveByPre(j, m int, suffix []int, prefix []bool) int {
   //后缀匹配长度
   l := m - 1 - j
   if suffix[l] != -1 {
      return j - suffix[l] + 1
   }
   num := m - 1
   for i := j + 2; i < num; i++ {
      if prefix[m-i] == true {
         return i
      }
   }
   return m
}
```

## **测试Code**

```
package string

import (
   "testing"
)

func TestBoyerMooreMatch(t *testing.T) {
   str1 := `毛选第五卷包括毛泽东在中华人民共和国建国后的文章共70篇，截至1957年。1977年4月15日由人民出版社正式出版发行，定价1.25元人民币。
本卷1967年开始筹备，原负责筹备工作的是中央文革小组理论组，以周恩来为名义上的负责人，不久即停止。1969年后，在康生的领导下再次开始准备工作，但不久后因未获得毛泽东的认可，工作又一次停止。[5]
毛泽东逝世后，华国锋决定继续出版毛选第五卷，在击溃四人帮后，于1976年10月8日再度开始筹备工作，由汪东兴为名义负责人，李鑫、吴泠西、胡绳、熊复实际负责编辑工作。
第五卷出版后，华国锋仍保留了毛泽东选集编辑委员会，准备筹备出版毛选第六卷（未出版），但随着邓小平的掌握政权，中共中央不再坚持对毛泽东的个人崇拜。1980年，毛泽东选集编辑委员会改组为中共中央文献研究室，此后未再出版毛选。[6]已发售的毛选第五卷，也在1982年后停售，目前只能在一些二手书刊交易平台购得。
为了最终弥补“毛选只有部分毛泽东文章，尤其没有建国后文章”的缺憾，中共中央在1990年代又组织编印了《毛泽东文集》、《毛泽东早期文稿》（内部发行）、《建国以来毛泽东文稿》（内部发行）等著作集。`
   str2 := "《毛泽东"
   str1Rune := []rune(str1)
   str2Rune := []rune(str2)
   res := BoyerMooreMatch(str1Rune, str2Rune, len(str1Rune), len(str2Rune))
   t.Log(res)
}
```