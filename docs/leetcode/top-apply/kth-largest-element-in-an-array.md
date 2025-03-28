# 数组中的第K个最大元素

## 题目

给定整数数组 `nums` 和整数 `k`，请返回数组中第 `k` 个最大的元素。

请注意，你需要找的是数组排序后的第 `k` 个最大的元素，而不是第 `k` 个不同的元素。

你必须设计并实现时间复杂度为 `O(n)` 的算法解决此问题。

示例:

```text
输入: [3,2,1,5,6,4], k = 2
输出: 5

输入: [3,2,3,1,2,4,5,5,6], k = 4
输出: 4
```

---

## code

```go
// Kth largest element in an array
func findKthLargest(nums []int, k int) int {
	l := len(nums)
	if k > l || k < 0 {
		return -1
	}
	// sort.Ints(nums)
	// MergeSort(nums)
	// QuickSort(nums)
	// return nums[l-k]
	// 堆排序
	heapSize := l
	// 构建大顶堆
	buildMaxHeap(nums, heapSize)
	for i := len(nums) - 1; i >= len(nums)-k+1; i-- {
		// 这里是交换最大最小值,来进行一次自顶向下的全堆化
		nums[0], nums[i] = nums[i], nums[0]
		// 这是缩减堆的大小,逼近所期望的第k大值
		heapSize--
		// 进行对话,堆顶是最大值
		maxHeapify(nums, 0, heapSize)
	}
	return nums[0]
}

func MergeSort(arr []int) {
	execMerge := func(arr []int, i, mid, j int) {
		tmpArr := make([]int, j-i+1)
		// 区间 [i,mid] 与 [mid+1,j] 的合并
		a, b, index := i, mid+1, 0
		for a <= mid && b <= j {
			if arr[a] <= arr[b] {
				tmpArr[index] = arr[a]
				a++
			} else {
				tmpArr[index] = arr[b]
				b++
			}
			index++
		}

		s, e := 0, -1
		if a <= mid {
			s, e = a, mid
		} else if b <= j {
			s, e = b, j
		}

		for k := s; k <= e && index < len(tmpArr); k, index = k+1, index+1 {
			tmpArr[index] = arr[k]
		}

		for k, a := i, 0; k <= j && a < len(tmpArr); k, a = k+1, a+1 {
			arr[k] = tmpArr[a]
		}

	}
	var execSort func(arr []int, i, j int)
	execSort = func(arr []int, i, j int) {
		// 终止条件
		if i >= j {
			return
		}

		// 取中位置
		mid := (i + j) / 2
		// 分治递归
		execSort(arr, i, mid)
		execSort(arr, mid+1, j)
		// 合并排序
		execMerge(arr, i, mid, j)
	}
	execSort(arr, 0, len(arr)-1)
}

func QuickSort(arr []int) {
	partition := func(arr []int, i, j int) int {
		pre, posVal := i, arr[j]
		for now := i; now < j; now++ {
			if arr[now] < posVal {
				arr[now], arr[pre] = arr[pre], arr[now]
				pre++
			}
		}
		arr[pre], arr[j] = arr[j], arr[pre]
		return pre
	}
	var execQuick func(arr []int, i, j int)
	execQuick = func(arr []int, i, j int) {
		if i >= j {
			return
		}
		// 进行切分,选取位置 pos 做分割 (i<=pos<=j)
		pos := partition(arr, i, j)
		execQuick(arr, i, pos-1)
		execQuick(arr, pos+1, j)

	}
	arr[0], arr[len(arr)-1] = arr[len(arr)-1], arr[0]
	execQuick(arr, 0, len(arr)-1)
}

func buildMaxHeap(a []int, heapSize int) {
	// 配置堆高
	for i := heapSize / 2; i >= 0; i-- {
		// 自顶向下执行堆化
		maxHeapify(a, i, heapSize)
	}
}

func maxHeapify(a []int, i, heapSize int) {
	// 初始化当前i的左右节点l ,r
	l, r, largest := i*2+1, i*2+2, i
	// 左节点是否大于当前节点值
	if l < heapSize && a[l] > a[largest] {
		largest = l
	}
	// 右节点是否大于当前节点值
	if r < heapSize && a[r] > a[largest] {
		largest = r
	}
	// 上述操作是为了保证数组堆化后左侧最大,右侧最小
	// 如果左右节点有大于当前节点的就开始堆化
	if largest != i {
		// 交换当前节点与大于其节点的左或右的位置
		a[i], a[largest] = a[largest], a[i]
		// 然后就可以再次堆化
		maxHeapify(a, largest, heapSize)
	}
}
```
