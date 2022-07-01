
# 回溯算法的初体验 八皇后 01背包 简单正则匹配

---

## **八皇后 Code**

```go
package backtrack

import "fmt"

//下标表示行,值表示queen存储在哪一列
var result [8]int

func Set8Queen(row int) {
   if row == 8 { //8个棋子都放置好了，打印结果
      printQueen()
      return
   }
   for column := 0; column < 8; column++ { //每一行对应8列结果
      if isOK(row, column) {
         result[row] = column
         Set8Queen(row + 1)
      }
   }
}

//判断row行column列放置是否合适
func isOK(row, column int) bool {
   leftUp, rightUp := column-1, column+1
   for i := row - 1; i >= 0; i-- { //逐行往上考察每一行
      //先判断第i行的column列是否放置
      if result[i] == column {
         return false
      }
      //判断左上对角线：第i行leftup列是否有放置
      if leftUp >= 0 && result[i] == leftUp {
         return false
      }
      //判断右上对角线：第i行rightup列是否有放置
      if rightUp < 8 && result[i] == rightUp {
         return false
      }
      leftUp--
      rightUp++
   }
   return true
}

func printQueen() {
   for row := 0; row < 8; row++ {
      for column := 0; column < 8; column++ {
         if result[row] == column {
            fmt.Print("Q ")
         } else {
            fmt.Print("* ")
         }
      }
      fmt.Println()
   }
   fmt.Println()
}
```

## **八皇后 测试Code**

```go
package greedy

import (
   "testing"
)

func TestNewHuffManTree(t *testing.T) {
   str := []rune(`对输入的英文大写字母进行统计概率 然后构建哈夫曼树,输出是按照概率降序排序输出Huffman编码`)
   h := NewHuffManTree(str)
   h.Print()
}
```

---

## **01背包 Code**

```go
package backtrack

var max int
/**
cw表示当前已经装进去的物品的重量和；index表示考察到哪个物品了；
bw背包重量；items表示每个物品的重量；n表示物品个数
假设背包可承受重量100，物品个数10，物品重量存储在数组a中，那可以这样调用函数：
f(0, 0, []int, 10, 100)
 */
func BagQuestion(index, cw int, items []int, n int, bw int) {
   if cw == bw || index == n { //背包满了或者物品放完
      if max < cw {
         max = cw
      }
      return
   }
   BagQuestion(index+1, cw, items, n, bw) //当前物品不装
   if cw+items[index] <= bw {
      BagQuestion(index+1, cw+items[index], items, n, bw) //当前物品装
   }
}

func GetMax() int {
   return max
}
```

## **01背包 测试Code**

```go
package backtrack

import "testing"

func TestBagQuestion(t *testing.T) {
   items := []int{
      5, 1, 9, 4,
   }
   BagQuestion(0, 0, items, len(items), 12)
   t.Log(GetMax())
}
```

---

## **简单小正则 匹配(0~n)或者?匹配(0~1) Code**

```go
package backtrack

type Pattern struct {
   matched bool
   pattern []rune //正则表达式
   plen    int    //正则表达式长度
}

func NewPattern(pattern []rune, plen int) *Pattern {
   return &Pattern{
      pattern: pattern,
      plen:    plen,
   }
}

func (p *Pattern) Match(text []rune, tlen int) bool {
   p.rematch(0, 0, text, tlen)
   return p.matched
}

func (p *Pattern) rematch(pIndex, tIndex int, text []rune, tlen int) {
   if p.matched { //标识已经匹配
      return
   }
   if pIndex == p.plen { //正则串已经到结尾
      if tIndex == tlen { //文本串已经匹配到结尾
         p.matched = true
      }
      return
   }
   if p.pattern[pIndex] == '*' { //匹配任意个字符(0~n)
      for i := 0; i < tlen-tIndex; i++ {
         p.rematch(pIndex+1, tIndex+i, text, tlen)
      }
   } else if p.pattern[pIndex] == '?' { //匹配0个或1个字符
      p.rematch(pIndex+1, tIndex, text, tlen)
      p.rematch(pIndex+1, tIndex+1, text, tlen)
   } else if tIndex < tlen && p.pattern[pIndex] == text[tIndex] { //纯字符匹配
      p.rematch(pIndex+1, tIndex+1, text, tlen)
   }
}
```

## **简单小正则 匹配(0~n)或者?匹配(0~1) 测试Code**

```go
package backtrack

import "testing"

func TestPattern_Match(t *testing.T) {
   patern := []rune("assh*666")
   text := []rune("asshsdfgdf的风格的发挥肺结核沙发上地方assh当时发生了来了bbb好666sjdkhfkjsdf的撒防守对方666")
   p := NewPattern(patern, len(patern))
   t.Log(p.Match(text, len(text)))
}
```
