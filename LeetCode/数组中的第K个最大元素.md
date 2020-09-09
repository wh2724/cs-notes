##  数组中的第K个最大元素

> 在未排序的数组中找到第 k 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。
>
> 示例 1:
>
> 输入: [3,2,1,5,6,4] 和 k = 2
> 输出: 5
> 示例 2:
>
> 输入: [3,2,3,1,2,4,5,5,6] 和 k = 4
> 输出: 4
> 说明:
>
> 你可以假设 k 总是有效的，且 1 ≤ k ≤ 数组的长度。
>
> 链接：https://leetcode-cn.com/problems/kth-largest-element-in-an-array



Python heapq docs:

> https://docs.python.org/zh-cn/3/library/heapq.html



```python
# 使用容量为K的小顶堆
import heapq

class Solution:
    def findKthLargest(self, nums: List[int], k: int) -> int:
        heap = nums[:k]
        heapq.heapify(heap)

        for num in nums[k:]:
            if num > heap[0]:
              	# heapq.heappop(heap) heapq.heappush(heap, num)
                heapq.heapreplace(heap, num)

        return heap[0]
      

# 利用快排思想: 快速排序一趟完成后，枢轴元素位置确定
class Solution:
    def findKthLargest(self, nums: List[int], k: int) -> int:
        size = len(nums)
        position = size - k

        def quick_sort(left, right):
            i, j = left, right
            pivot = nums[left]
            while i < j:
                while i < j and nums[j] >= pivot:
                    j -= 1
                nums[i] = nums[j]

                while i < j and nums[i] < pivot:
                    i += 1
                nums[j] = nums[i] 
            
            nums[i] = pivot
            if i == position:
                return pivot
            elif i < position:
                return quick_sort(i + 1, right)
            else:
                return quick_sort(left, i - 1)

        return quick_sort(0, size - 1)
```

