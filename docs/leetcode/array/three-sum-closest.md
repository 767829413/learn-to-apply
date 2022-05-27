# 最接近的三数之和

## 题目

给你一个长度为 n 的整数数组 nums 和 一个目标值 target

请你从 nums 中选出三个整数，使它们的和与 target 最接近

假定每组输入只存在恰好一个解

示例:

```text
输入：nums = [-1,2,1,-4], target = 1
输出：2
解释：与 target 最接近的和是 2 (-1 + 2 + 1 = 2)
```

---

## code

```go
func threeSumClosest(nums []int, target int) int {
 res := math.MaxInt
 l := len(nums)
 //特判,数组长度小于3就返回
 if l < 3 {
  return 0
 }
 // 排序,从小到大
 sort.Ints(nums)
 // 遍历排序后的数组
 for i, _ := range nums {
  // 过滤重复的值
  if i > 0 && nums[i] == nums[i-1] {
   continue
  }
  lo := i + 1
  hi := l - 1
  for lo < hi {
   r := nums[i] + nums[lo] + nums[hi]
   tmp := r - target
   if tmp == 0 {
    return r
   } else if r > target { // 和大于 target,那么右边界向左,总和缩小
    // 过滤重复的值
    for lo < hi && nums[hi] == nums[hi-1] {
     hi--
    }
    hi--
   } else { // 和小于 target,那么左边界向右,总和增大
    // 过滤重复的值
    for lo < hi && nums[lo] == nums[lo-1] {
     lo++
    }
    lo++
   }
   res = approachNumber(res, r, target)
  }
 }
 return res
}

func approachNumber(a, b, c int) int {
 l := math.Abs(float64(a) - float64(c))
 r := math.Abs(float64(b) - float64(c))
 if l > r {
  return b
 }
 return a
}

```
