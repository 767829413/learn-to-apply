# 四数之和

## 题目

给你一个由 n 个整数组成的数组 nums ，和一个目标值 target

请你找出并返回满足下述全部条件且不重复的四元组

 **[nums[a], nums[b], nums[c], nums[d]]**

注意:（若两个四元组元素一一对应，则认为两个四元组重复）

* 0 <= a, b, c, d < n
* a、b、c 和 d 互不相同
* nums[a] + nums[b] + nums[c] + nums[d] == target

你可以按 任意顺序 返回答案

示例:

```text
输入：nums = [1,0,-1,0,-2,2], target = 0
输出：[[-2,-1,1,2],[-2,0,0,2],[-1,0,0,1]]
```

---

## code

```go
func fourSum(nums []int, target int) [][]int {
 res := [][]int{}
 l := len(nums)
 //特判,数组长度小于4就返回
 if l < 4 {
  return res
 }
 // 排序,从小到大
 sort.Ints(nums)
 // 遍历排序后的数组
 for i := 0; i < l-3; i++ {
  if i > 0 && nums[i] == nums[i-1] {
   continue
  }
  // 第一个数固定了,从小到大的顺序相加,只要和大于target,则后面的不可能出现等于target了
  if nums[i]+nums[i+1]+nums[i+2]+nums[i+3] > target {
   return res
  }
  // 第一个数固定了,从大到小的顺序相加,只要和小于 target,第一个数和剩下的任意三个数相加都会小于,直接跳过
  if nums[i]+nums[l-3]+nums[l-2]+nums[l-1] < target {
   continue
  }
  for j := i + 1; j < l-2; j++ {
   if j > i+1 && nums[j] == nums[j-1] {
    continue
   }
   // 前两个数固定了,从小到大的顺序相加,只要和大于target,则后面的不可能出现等于target了
   if nums[i]+nums[j]+nums[j+1]+nums[j+2] > target {
    break
   }
   // 前两个数固定了,从大到小的顺序相加,只要和小于 target,和剩下的任意两个数相加都会小于,直接跳过
   if nums[i]+nums[j]+nums[l-2]+nums[l-1] < target {
    continue
   }
   lo := j + 1
   hi := l - 1
   for lo < hi {
    tmp := nums[i] + nums[j] + nums[lo] + nums[hi]
    if tmp == target {
     res = append(res, []int{nums[i], nums[j], nums[lo], nums[hi]})
     //重复值跳过
     for lo < hi && nums[lo] == nums[lo+1] {
      lo++
     }
     //重复值跳过
     for lo < hi && nums[hi] == nums[hi-1] {
      hi--
     }
     lo++
     hi--
    } else if tmp > target { // 和大于 target,那么右边界向左,总和缩小
     hi--
    } else { // 和小于 target,那么左边界向右,总和增大
     lo++
    }
   }
  }
 }
 return res
}

```
