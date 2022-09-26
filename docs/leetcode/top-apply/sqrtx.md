# x 的平方根

## 题目

给你一个非负整数 x ，计算并返回 x 的 算术平方根 。

由于返回类型是整数，结果只保留 整数部分 ，小数部分将被 舍去 。

注意：不允许使用任何内置指数函数和算符，例如 pow(x, 0.5) 或者 x ** 0.5 。

示例:

```text
输入：x = 4
输出：2

输入：x = 8
输出：2
解释：8 的算术平方根是 2.82842..., 由于返回类型是整数，小数部分将被舍去。
```

---

## code

```go
// Sqrtx
// 牛顿迭代或者二分查找
func mySqrt(x int) int {
	if false {
		s := 1.0
		target := float64(x)
		for {
			tmp := (s + target/s) / 2.0
			if math.Abs(tmp-s) <= 0.1 {
				s = tmp
				break
			}
			s = tmp
		}
		return int(math.Floor(s))
	} else {
		ans, low, high := 0, 0, x
		for low <= high {
			mid := low + (high-low)/2
			if mid*mid <= x {
				ans = mid
				low = mid + 1
			} else {
				high = mid - 1
			}
		}
		return ans
	}
}
```
