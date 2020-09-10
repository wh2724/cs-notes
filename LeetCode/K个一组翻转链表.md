## K个一组翻转链表

> 给你一个链表，每 k 个节点一组进行翻转，请你返回翻转后的链表。
>
> k 是一个正整数，它的值小于或等于链表的长度。
>
> 如果节点总数不是 k 的整数倍，那么请将最后剩余的节点保持原有顺序。
>
>  
>
> 示例：
>
> 给你这个链表：1->2->3->4->5
>
> 当 k = 2 时，应当返回: 2->1->4->3->5
>
> 当 k = 3 时，应当返回: 3->2->1->4->5
>
>
> 链接：https://leetcode-cn.com/problems/reverse-nodes-in-k-group

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution:
    def reverseKGroup(self, head: ListNode, k: int) -> ListNode:
        if not head:
            return None

        node = cur = head
        satisfy_k = True
        for _ in range(k):
            if node:
                node = node.next
            else:
                satisfy_k = False

        if not satisfy_k:
            return head

        pre = self.reverseKGroup(node, k)
        for _ in range(k):
            cur.next, pre, cur = pre, cur, cur.next
        
        return pre
```

