# 三数之和

## 题目

给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有和为 0 且不重复的三元组

注意：答案中不可以包含重复的三元组

示例:

```text
输入：nums = [-1,0,1,2,-1,-4]
输出：[[-1,-1,2],[-1,0,1]]
```

---

## code

```go
func threeSum(nums []int) [][]int {
 var res [][]int
 l := len(nums)
 //特判,数组长度小于3就返回
 if l < 3 {
  return res
 }
 // 排序,从小到大
 sort.Ints(nums)
 // 遍历排序后的数组
 for i, _ := range nums {
  // 已经排好序,当nums[i]>0时.后面不会出现和为零了
  if nums[i] > 0 {
   return res
  }
  // 过滤重复的值
  if i > 0 && nums[i] == nums[i-1] {
   continue
  }
  lo := i + 1
  hi := l - 1
  for lo < hi {
   r := nums[i] + nums[lo] + nums[hi]
   if r == 0 {
    res = append(res, []int{nums[i], nums[lo], nums[hi]})
    // 过滤重复的值
    for lo < hi && nums[lo] == nums[lo+1] {
     lo++
    }
    // 过滤重复的值
    for lo < hi && nums[hi] == nums[hi-1] {
     hi--
    }
    hi--
    lo++
   } else if r > 0 { // 和大于0,那么右边界向左,总和缩小
    hi--
   } else { // 和小于0,那么左边界向右,总和增大
    lo++
   }
  }
 }
 return res
}

```
