
# 简单Trie树的实现

## **演示Code**
```
package string

type trieNode struct {
   s         rune
   children  map[rune]*trieNode
   isEndChar bool
}

type Trie struct {
   Root *trieNode
}

func NewTrie() *Trie {
   return &Trie{
      Root: &trieNode{
         s:        rune('/'),
         children: make(map[rune]*trieNode),
      },
   }
}

func (t *Trie) Insert(text []rune) {
   cur := t.Root
   num := len(text)
   for i := 0; i < num; i++ {
      if v, ok := cur.children[text[i]]; ok {
         cur = v
      } else {
         cur.children[text[i]] = &trieNode{
            s:        text[i],
            children: make(map[rune]*trieNode),
         }
         cur = cur.children[text[i]]
      }
   }
   cur.isEndChar = true
}

func (t *Trie) Find(text []rune) bool {
   cur := t.Root
   num := len(text)
   for i := 0; i < num; i++ {
      if v, ok := cur.children[text[i]]; ok {
         cur = v
      }
   }
   if !cur.isEndChar {
      return false
   }
   return true
}
```

## **测试Code**

```
package string

import (
   "testing"
)

func TestTrie(t *testing.T) {
   testStrs := []string{
      "人生如初见",
      "人就考很快乐回事",
      "xcfs;xflgksdkg;lk",
      "ewuieywruyrwhm学科划分SD卡",
   }
   tt := NewTrie()
   for _, v := range testStrs {
      tt.Insert([]rune(v))
   }
   for i := len(testStrs) - 1; i >= 0; i-- {
      t.Log(tt.Find([]rune(testStrs[i])))
   }
}
```

