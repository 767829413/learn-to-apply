# 下一个排列

## 题目

整数数组 **nums** 按升序排列，数组中的值 **互不相同** 。

在传递给函数之前，**nums** 在预先未知的某个下标 **k（0 <= k < nums.length）** 上进行了 **旋转** ，使数组变为 **[nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]（下标 从 0 开始 计数）** 。例如， **[0,1,2,4,5,6,7]** 在下标 **3** 处经旋转后可能变为 **[4,5,6,7,0,1,2]** 。

给你 **旋转后** 的数组 **nums** 和一个整数 **target** ，如果 **nums** 中存在这个目标值 **target** ，则返回它的下标，否则返回 **-1** 。

示例:

```text
输入：nums = [4,5,6,7,0,1,2], target = 0
输出：4
```

---

## code

```go
func search(nums []int, target int) int {
	l := len(nums)
	if l == 0 {
		return -1
	}
	if l == 1 {
		if nums[0] == target {
			return 0
		} else {
			return -1
		}
	}
	// 初始化边界和循环退出条件
	for lo, hi := 0, l-1; lo <= hi; {
		mid := (lo + hi) >> 1
		if nums[mid] == target {
			return mid
		}
		// 数组数据二分,不包含旋转点的一定是正常升序
		// 目标在正常升序中,继续二分即可
		// 目标在有旋转点的部分,可以继续二分,重新判断按照上述逻辑判断
		if nums[0] <= nums[mid] { // 表示[0,mid)正常升序的,不包含旋转点,常规二分
			if nums[0] <= target && target < nums[mid] {
				hi = mid - 1
			} else { // 重新定义,mid值随之更新
				lo = mid + 1
			}
		} else { // [0,mid)包含旋转点,那么[mid,l)为正常升序,常规二分
			if nums[mid] < target && target <= nums[l-1] {
				lo = mid + 1
			} else { // 重新定义,mid值随之更新
				hi = mid - 1
			}
		}
	}
	return -1
}
```
