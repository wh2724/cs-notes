## 复原IP地址

> 给定一个只包含数字的字符串，复原它并返回所有可能的 IP 地址格式。
>
> 有效的 IP 地址 正好由四个整数（每个整数位于 0 到 255 之间组成，且不能含有前导 0），整数之间用 '.' 分隔。
>
> 例如："0.1.2.201" 和 "192.168.1.1" 是 有效的 IP 地址，但是 "0.011.255.245"、"192.168.1.312" 和 "192.168@1.1" 是 无效的 IP 地址。
>
> 示例 1：
>
> 输入：s = "25525511135"
> 输出：["255.255.11.135","255.255.111.35"]
> 示例 2：
>
> 输入：s = "0000"
> 输出：["0.0.0.0"]
> 示例 3：
>
> 输入：s = "1111"
> 输出：["1.1.1.1"]
> 示例 4：
>
> 输入：s = "010010"
> 输出：["0.10.0.10","0.100.1.0"]
> 示例 5：
>
> 输入：s = "101023"
> 输出：["1.0.10.23","1.0.102.3","10.1.0.23","10.10.2.3","101.0.2.3"]
>
> 链接：https://leetcode-cn.com/problems/restore-ip-addresses

```python
class Solution:
    def restoreIpAddresses(self, s: str) -> List[str]:
        res = []

        def backtrack(cur, remian_s, count):
            if count == 4:
                if remian_s == '':
                    res.append(cur)
                return

            if count > 0:
                cur += '.'

            length = len(remian_s)
            if length > 0:
                backtrack(cur+remian_s[0], remian_s[1:], count+1)
            if length > 1 and remian_s[0] != '0':
                backtrack(cur+remian_s[0:2], remian_s[2:], count+1)
            if length > 2 and 99 < int(remian_s[0:3]) < 256:
                backtrack(cur+remian_s[0:3], remian_s[3:], count+1)
        
        backtrack('', s, 0)

        return res
```

