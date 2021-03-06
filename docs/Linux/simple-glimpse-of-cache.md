
# 初窥cache一角

## **code**

```go
func main() {
   arr1 := [10000][10000]int{}
   arr2 := [10000][10000]int{}
   start1 := time.Now()
   for i := 0; i < 10000; i++ {
      for j := 0; j < 10000; j++ {
         arr1[i][j] += 1
      }
   }
   fmt.Println(time.Since(start1))
   start2 := time.Now()
   for i := 0; i < 10000; i++ {
      for j := 0; j < 10000; j++ {
         arr2[j][i] += 1
      }
   }
   fmt.Println(time.Since(start2))
}
```

## **运行的结果是**

```bash
    218.0312ms
    1.2320012s
```

## **稍微总结下**

1. **数组的存储方式是连续的，在内存中应该是按照行来存储的，遍历的时候也是一个一个的往后遍历**

2. **根据资料得知,CPU从内存读取数据到CPU Cache ，是按照块的方式读取的，在物理内存上必定是连续的,那么可以推测就是一行一行的存储的，此时采用按列来迭代的话,会导致CPU Cache大量失效(未命中),只能重新从内存中加载,众所周知CPU Cache 和 内存的访问性能差异是及其巨大的,照这个理解,按行来迭代处理数据就能最大化的利用CPU Cache,从而提高性能**
