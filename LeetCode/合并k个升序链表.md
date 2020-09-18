## 合并K个升序链表

> 给你一个链表数组，每个链表都已经按升序排列。
>
> 请你将所有链表合并到一个升序链表中，返回合并后的链表。
>
> 示例 1：
>
> 输入：lists = [[1,4,5],[1,3,4],[2,6]]
> 输出：[1,1,2,3,4,4,5,6]
> 解释：链表数组如下：
> [
>   1->4->5,
>   1->3->4,
>   2->6
> ]
> 将它们合并到一个有序链表中得到。
> 1->1->2->3->4->4->5->6
>
> 示例 2：
>
> 输入：lists = []
> 输出：[]
>
> 示例 3：
>
> 输入：lists = [[]]
> 输出：[]
>
> 提示：
>
> k == lists.length
> 0 <= k <= 10^4
> 0 <= lists[i].length <= 500
> -10^4 <= lists[i][j] <= 10^4
> lists[i] 按 升序 排列
> lists[i].length 的总和不超过 10^4
>
> 链接：https://leetcode-cn.com/problems/merge-k-sorted-lists



Python heapq docs:

> https://docs.python.org/zh-cn/3/library/heapq.html



```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

# 两两合并，会超时
def merge_K_lists(self, lists: List[ListNode]) -> ListNode:
    def merge_two_lists(l1, l2):
    	  if l1 and l2:
            if l1.val > l2.val:
                l1, l2 = l2, l1
            l1.next = merge_two_lists(l1.next, l2)
        return l1 or l2

    if not lists:
        return None

    res = lists[0]
    for i in range(1, len(lists)):
    res = merge_two_lists(res, lists[i])
	
    return res

# 使用Python内置堆heapq
import heapq
def merge_K_lists(self, lists: List[ListNode]) -> ListNode:
    pre = ListNode(0)
    cur = pre

    # init heap
    heap = []
    for index, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, index))

    while heap:
        _val, index = heapq.heappop(heap)
        node = lists[index]
        cur.next = node
        cur = cur.next

        if node.next:
            lists[index] = node.next
            heapq.heappush(heap, (node.next.val, index))
      
    return pre.next
  
```

