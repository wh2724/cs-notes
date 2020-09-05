## 二叉树的锯齿形层次遍历(Z字遍历/之字遍历)

> 给定一个二叉树，返回其节点值的锯齿形层次遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。
>
> 例如：
> 给定二叉树 [3,9,20,null,null,15,7],
>
> ​	3
>
>    / \
>   9  20
>       /  \
>    15   7
> 返回锯齿形层次遍历如下：
>
> [
>   [3],
>   [20,9],
>   [15,7]
> ]
>
>
> 链接：https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal

```python
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, x):
#         self.val = x
#         self.left = None
#         self.right = None

class Solution:
    def zigzagLevelOrder(self, root: TreeNode) -> List[List[int]]:
        if not root:
            return []

        que = [root]
        res = []
        level = 0
        
        while que:
            level += 1
            cur_level = []
            next_level_node = []
            for node in que:
                cur_level.append(node.val)
                if node.left:
                    next_level_node.append(node.left)
                if node.right:
                    next_level_node.append(node.right)
            
            # 偶数层翻转
            res.append(cur_level) if level & 1 else res.append(cur_level[::-1])      
            que = next_level_node

        return res
```

