## LRU Cache

> 运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。
>
> 获取数据 get(key) - 如果关键字 (key) 存在于缓存中，则获取关键字的值（总是正数），否则返回 -1。
> 写入数据 put(key, value) - 如果关键字已经存在，则变更其数据值；如果关键字不存在，则插入该组「关键字/值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。
>
> 示例:
>
> LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );
>
> cache.put(1, 1);
> cache.put(2, 2);
> cache.get(1);       // 返回  1
> cache.put(3, 3);    // 该操作会使得关键字 2 作废
> cache.get(2);       // 返回 -1 (未找到)
> cache.put(4, 4);    // 该操作会使得关键字 1 作废
> cache.get(1);       // 返回 -1 (未找到)
> cache.get(3);       // 返回  3
> cache.get(4);       // 返回  4
>
> 链接：https://leetcode-cn.com/problems/lru-cache



#### Python 官方文档推荐推荐实现：

> https://docs.python.org/zh-cn/3/library/collections.html#collections.OrderedDict

```python
from collections import OrderedDict

class LRU(OrderedDict):
    'Limit size, evicting the least recently looked-up key when full'

    def __init__(self, maxsize=128, /, *args, **kwds):
        self.maxsize = maxsize
        super().__init__(*args, **kwds)

    def __getitem__(self, key):
        value = super().__getitem__(key)
        self.move_to_end(key)
        return value

    def __setitem__(self, key, value):
        if key in self:
            self.move_to_end(key)
        super().__setitem__(key, value)
        if len(self) > self.maxsize:
            oldest = next(iter(self))
            del self[oldest]
            # self.popitem(last=False)

```



```python
from collections import OrderedDict

class LRUCache:

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.order_dict = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.order_dict:
            return -1
        self.order_dict.move_to_end(key)
        return self.order_dict[key]

    def put(self, key: int, value: int) -> None:
        self.order_dict[key] = value
        self.order_dict.move_to_end(key)
        if len(self.order_dict) > self.capacity:
            self.order_dict.popitem(last=False)
```

