俏贫堆净嘲



============================================================
 Python LRU缓存实现 — OrderedDict/手动双链表/functools/线程安全
============================================================

LRU（Least Recently Used）缓存淘汰最近最少使用的数据，是缓存算法的经典实现。
Python提供了多种方式实现LRU缓存，从内置装饰器到手写数据结构。

============================================================
1. 基于 OrderedDict 的 LRU 缓存
============================================================

from collections import OrderedDict

class LRUCacheOrderedDict:
    """使用OrderedDict实现LRU缓存，简洁高效"""
    def __init__(self, capacity):
        self.capacity = capacity          # 缓存容量
        self.cache = OrderedDict()        # 有序字典保持访问顺序

    def get(self, key):
        """获取键对应的值，若不存在返回-1"""
        if key not in self.cache:
            return -1
        # 将访问的键移到末尾（表示最近使用）
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        """插入或更新键值对"""
        if key in self.cache:
            # 已存在：更新值并移到末尾
            self.cache.move_to_end(key)
        elif len(self.cache) >= self.capacity:
            # 缓存已满：弹出最久未使用的（第一个元素）
            self.cache.popitem(last=False)
        self.cache[key] = value

    def __repr__(self):
        return f"LRUCache({dict(self.cache)})"

# 测试
lru = LRUCacheOrderedDict(3)
lru.put('A', 1); lru.put('B', 2); lru.put('C', 3)
print(lru)                               # {'A':1, 'B':2, 'C':3}
lru.get('A')                             # 访问A，A移到末尾
lru.put('D', 4)                          # 新增D，淘汰最久未用的B
print("淘汰B后:", lru)                   # {'C':3, 'A':1, 'D':4}

============================================================
2. 手动实现：双向链表 + 字典
============================================================

class DLinkedNode:
    """双向链表节点，用于手动LRU"""
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCacheManual:
    """手写双向链表+字典实现LRU，面试常考"""
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}                   # {key: DLinkedNode}
        # 虚拟头尾节点简化边界处理
        self.head = DLinkedNode()         # 最近使用的
        self.tail = DLinkedNode()         # 最久未使用的
        self.head.next = self.tail
        self.tail.prev = self.head

    def _add_node(self, node):
        """将节点添加到头部（最近使用位置）"""
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove_node(self, node):
        """从链表中移除一个节点"""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _move_to_head(self, node):
        """将节点移动到头部（表示刚被访问）"""
        self._remove_node(node)
        self._add_node(node)

    def _pop_tail(self):
        """弹出尾部节点（最久未使用的）"""
        node = self.tail.prev
        self._remove_node(node)
        return node

    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._move_to_head(node)          # 标记为最近使用
        return node.value

    def put(self, key, value):
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._move_to_head(node)
        else:
            if len(self.cache) >= self.capacity:
                tail = self._pop_tail()   # 淘汰最久未使用的
                del self.cache[tail.key]
            new_node = DLinkedNode(key, value)
            self.cache[key] = new_node
            self._add_node(new_node)

============================================================
3. functools.lru_cache 装饰器
============================================================

from functools import lru_cache
import time

@lru_cache(maxsize=128)
def fibonacci(n):
    """带缓存的斐波那契数列计算"""
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# lru_cache的参数：
# - maxsize: 最大缓存条目数（设为None表示无上限）
# - typed: 是否区分不同类型（如3和3.0）

# 第一次调用会递归计算，后续调用直接返回缓存结果
start = time.time()
result = fibonacci(35)
print(f"fib(35) = {result}, 耗时: {time.time() - start:.4f}s")

# 查看缓存统计信息
print("缓存统计:", fibonacci.cache_info())
# CacheInfo(hits=..., misses=..., maxsize=128, currsize=...)

# 清空缓存
fibonacci.cache_clear()

# 使用lru_cache实现函数结果的自动过期并不是内置支持
# 但对于固定输入输出的纯函数，lru_cache是非常好用的工具

============================================================
4. LRU vs LFU 对比
============================================================
# LRU（最近最少使用）：
#   - 淘汰最久未被访问的条目
#   - 实现相对简单
#   - 对突发性大量访问敏感（可能淘汰真正常用的数据）
#
# LFU（最不经常使用）：
#   - 淘汰访问频率最低的条目
#   - 需要维护访问计数
#   - 对历史访问模式有"记忆"
#
# 选择建议：
#   - 数据访问模式相对稳定 → LRU
#   - 有明显的热点数据和冷门数据 → LFU

============================================================
5. 基于时间的过期缓存
============================================================

import time as time_module

class ExpiringCache:
    """带过期时间的缓存"""
    def __init__(self, default_ttl=60):
        self.cache = {}                   # {key: (value, expire_time)}
        self.default_ttl = default_ttl    # 默认过期时间（秒）

    def get(self, key):
        if key not in self.cache:
            return None
        value, expire = self.cache[key]
        if time_module.time() > expire:   # 已过期
            del self.cache[key]
            return None
        return value

    def set(self, key, value, ttl=None):
        expire = time_module.time() + (ttl or self.default_ttl)
        self.cache[key] = (value, expire)

    def cleanup(self):
        """清理所有过期条目"""
        now = time_module.time()
        expired = [k for k, (_, t) in self.cache.items() if now > t]
        for k in expired:
            del self.cache[k]

============================================================
6. 大小受限缓存（Size-Bounded Cache）
============================================================

class SizeBoundedCache:
    """限制内容总大小的缓存（按字节计数）"""
    def __init__(self, max_size_bytes=1024*1024):
        self.cache = OrderedDict()
        self.max_size = max_size_bytes
        self.current_size = 0

    def get(self, key):
        if key not in self.cache:
            return None
        self.cache.move_to_end(key)
        return self.cache[key][0]

    def set(self, key, value):
        value_size = len(str(value))      # 估算大小
        if key in self.cache:
            old_value, old_size = self.cache[key]
            self.current_size -= old_size
        # 如果单个值就超过缓存限制，拒绝缓存
        if value_size > self.max_size:
            return
        # 腾出空间
        while self.current_size + value_size > self.max_size:
            _, (_, removed_size) = self.cache.popitem(last=False)
            self.current_size -= removed_size
        self.cache[key] = (value, value_size)
        self.current_size += value_size

============================================================
7. 线程安全的LRU缓存
============================================================

from threading import Lock

class ThreadSafeLRUCache:
    """使用Lock保证线程安全的LRU缓存"""
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = Lock()                # 保证线程安全

    def get(self, key):
        with self.lock:                   # 加锁保护临界区
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)
            return self.cache[key]

    def put(self, key, value):
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            elif len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)
            self.cache[key] = value

思赵啃恼迷诙抛肪恼窍肪八侥考聪

m.wzt.kxnxh.cn/02844.Doc
m.wzt.kxnxh.cn/73715.Doc
m.wzt.kxnxh.cn/46022.Doc
m.wzt.kxnxh.cn/88406.Doc
m.wzt.kxnxh.cn/93739.Doc
m.wzt.kxnxh.cn/93911.Doc
m.wzt.kxnxh.cn/28088.Doc
m.wzt.kxnxh.cn/11339.Doc
m.wzt.kxnxh.cn/13951.Doc
m.wzt.kxnxh.cn/80228.Doc
m.wzt.kxnxh.cn/59953.Doc
m.wzt.kxnxh.cn/31939.Doc
m.wzt.kxnxh.cn/82462.Doc
m.wzt.kxnxh.cn/35511.Doc
m.wzt.kxnxh.cn/37759.Doc
m.wzr.kxnxh.cn/82600.Doc
m.wzr.kxnxh.cn/22088.Doc
m.wzr.kxnxh.cn/51919.Doc
m.wzr.kxnxh.cn/11575.Doc
m.wzr.kxnxh.cn/00864.Doc
m.wzr.kxnxh.cn/24620.Doc
m.wzr.kxnxh.cn/20462.Doc
m.wzr.kxnxh.cn/99179.Doc
m.wzr.kxnxh.cn/68240.Doc
m.wzr.kxnxh.cn/99555.Doc
m.wzr.kxnxh.cn/64048.Doc
m.wzr.kxnxh.cn/91173.Doc
m.wzr.kxnxh.cn/75355.Doc
m.wzr.kxnxh.cn/77119.Doc
m.wzr.kxnxh.cn/44848.Doc
m.wzr.kxnxh.cn/91951.Doc
m.wzr.kxnxh.cn/42824.Doc
m.wzr.kxnxh.cn/37537.Doc
m.wzr.kxnxh.cn/24628.Doc
m.wzr.kxnxh.cn/53575.Doc
m.wze.kxnxh.cn/39177.Doc
m.wze.kxnxh.cn/88282.Doc
m.wze.kxnxh.cn/57393.Doc
m.wze.kxnxh.cn/24624.Doc
m.wze.kxnxh.cn/91799.Doc
m.wze.kxnxh.cn/17579.Doc
m.wze.kxnxh.cn/59195.Doc
m.wze.kxnxh.cn/19157.Doc
m.wze.kxnxh.cn/93755.Doc
m.wze.kxnxh.cn/48428.Doc
m.wze.kxnxh.cn/48886.Doc
m.wze.kxnxh.cn/40244.Doc
m.wze.kxnxh.cn/02466.Doc
m.wze.kxnxh.cn/31179.Doc
m.wze.kxnxh.cn/11355.Doc
m.wze.kxnxh.cn/71955.Doc
m.wze.kxnxh.cn/40800.Doc
m.wze.kxnxh.cn/37757.Doc
m.wze.kxnxh.cn/99537.Doc
m.wze.kxnxh.cn/64242.Doc
m.wzw.kxnxh.cn/71151.Doc
m.wzw.kxnxh.cn/35355.Doc
m.wzw.kxnxh.cn/28684.Doc
m.wzw.kxnxh.cn/82286.Doc
m.wzw.kxnxh.cn/64208.Doc
m.wzw.kxnxh.cn/04082.Doc
m.wzw.kxnxh.cn/48006.Doc
m.wzw.kxnxh.cn/86042.Doc
m.wzw.kxnxh.cn/39315.Doc
m.wzw.kxnxh.cn/06680.Doc
m.wzw.kxnxh.cn/79373.Doc
m.wzw.kxnxh.cn/80664.Doc
m.wzw.kxnxh.cn/82824.Doc
m.wzw.kxnxh.cn/57555.Doc
m.wzw.kxnxh.cn/15977.Doc
m.wzw.kxnxh.cn/55313.Doc
m.wzw.kxnxh.cn/60624.Doc
m.wzw.kxnxh.cn/19159.Doc
m.wzw.kxnxh.cn/04682.Doc
m.wzw.kxnxh.cn/82808.Doc
m.wzq.kxnxh.cn/28226.Doc
m.wzq.kxnxh.cn/59931.Doc
m.wzq.kxnxh.cn/11511.Doc
m.wzq.kxnxh.cn/06488.Doc
m.wzq.kxnxh.cn/22282.Doc
m.wzq.kxnxh.cn/44222.Doc
m.wzq.kxnxh.cn/42082.Doc
m.wzq.kxnxh.cn/28042.Doc
m.wzq.kxnxh.cn/75531.Doc
m.wzq.kxnxh.cn/04484.Doc
m.wzq.kxnxh.cn/13739.Doc
m.wzq.kxnxh.cn/97599.Doc
m.wzq.kxnxh.cn/13517.Doc
m.wzq.kxnxh.cn/31917.Doc
m.wzq.kxnxh.cn/48004.Doc
m.wzq.kxnxh.cn/35779.Doc
m.wzq.kxnxh.cn/93959.Doc
m.wzq.kxnxh.cn/73917.Doc
m.wzq.kxnxh.cn/80662.Doc
m.wzq.kxnxh.cn/66268.Doc
m.wlm.kxnxh.cn/55195.Doc
m.wlm.kxnxh.cn/39519.Doc
m.wlm.kxnxh.cn/79119.Doc
m.wlm.kxnxh.cn/93977.Doc
m.wlm.kxnxh.cn/04460.Doc
m.wlm.kxnxh.cn/73159.Doc
m.wlm.kxnxh.cn/42844.Doc
m.wlm.kxnxh.cn/62484.Doc
m.wlm.kxnxh.cn/22284.Doc
m.wlm.kxnxh.cn/68086.Doc
m.wlm.kxnxh.cn/11755.Doc
m.wlm.kxnxh.cn/26824.Doc
m.wlm.kxnxh.cn/91797.Doc
m.wlm.kxnxh.cn/57171.Doc
m.wlm.kxnxh.cn/91531.Doc
m.wlm.kxnxh.cn/66224.Doc
m.wlm.kxnxh.cn/24826.Doc
m.wlm.kxnxh.cn/13731.Doc
m.wlm.kxnxh.cn/51515.Doc
m.wlm.kxnxh.cn/88864.Doc
m.wln.kxnxh.cn/46860.Doc
m.wln.kxnxh.cn/93593.Doc
m.wln.kxnxh.cn/11737.Doc
m.wln.kxnxh.cn/84880.Doc
m.wln.kxnxh.cn/68000.Doc
m.wln.kxnxh.cn/75933.Doc
m.wln.kxnxh.cn/75735.Doc
m.wln.kxnxh.cn/17775.Doc
m.wln.kxnxh.cn/15175.Doc
m.wln.kxnxh.cn/40004.Doc
m.wln.kxnxh.cn/73195.Doc
m.wln.kxnxh.cn/04064.Doc
m.wln.kxnxh.cn/31519.Doc
m.wln.kxnxh.cn/33337.Doc
m.wln.kxnxh.cn/33937.Doc
m.wln.kxnxh.cn/71979.Doc
m.wln.kxnxh.cn/51935.Doc
m.wln.kxnxh.cn/55731.Doc
m.wln.kxnxh.cn/35179.Doc
m.wln.kxnxh.cn/80048.Doc
m.wlb.kxnxh.cn/55317.Doc
m.wlb.kxnxh.cn/31375.Doc
m.wlb.kxnxh.cn/93979.Doc
m.wlb.kxnxh.cn/24822.Doc
m.wlb.kxnxh.cn/40022.Doc
m.wlb.kxnxh.cn/37353.Doc
m.wlb.kxnxh.cn/82048.Doc
m.wlb.kxnxh.cn/95533.Doc
m.wlb.kxnxh.cn/31153.Doc
m.wlb.kxnxh.cn/08826.Doc
m.wlb.kxnxh.cn/24082.Doc
m.wlb.kxnxh.cn/35919.Doc
m.wlb.kxnxh.cn/13737.Doc
m.wlb.kxnxh.cn/73917.Doc
m.wlb.kxnxh.cn/73759.Doc
m.wlb.kxnxh.cn/91755.Doc
m.wlb.kxnxh.cn/84026.Doc
m.wlb.kxnxh.cn/55577.Doc
m.wlb.kxnxh.cn/60268.Doc
m.wlb.kxnxh.cn/39777.Doc
m.wlv.kxnxh.cn/97775.Doc
m.wlv.kxnxh.cn/93757.Doc
m.wlv.kxnxh.cn/20080.Doc
m.wlv.kxnxh.cn/82866.Doc
m.wlv.kxnxh.cn/53111.Doc
m.wlv.kxnxh.cn/08488.Doc
m.wlv.kxnxh.cn/60602.Doc
m.wlv.kxnxh.cn/44682.Doc
m.wlv.kxnxh.cn/59199.Doc
m.wlv.kxnxh.cn/79397.Doc
m.wlv.kxnxh.cn/31173.Doc
m.wlv.kxnxh.cn/95991.Doc
m.wlv.kxnxh.cn/02486.Doc
m.wlv.kxnxh.cn/57379.Doc
m.wlv.kxnxh.cn/13319.Doc
m.wlv.kxnxh.cn/64282.Doc
m.wlv.kxnxh.cn/86202.Doc
m.wlv.kxnxh.cn/64266.Doc
m.wlv.kxnxh.cn/37737.Doc
m.wlv.kxnxh.cn/17557.Doc
m.wlc.kxnxh.cn/59931.Doc
m.wlc.kxnxh.cn/64280.Doc
m.wlc.kxnxh.cn/42886.Doc
m.wlc.kxnxh.cn/59373.Doc
m.wlc.kxnxh.cn/53953.Doc
m.wlc.kxnxh.cn/37997.Doc
m.wlc.kxnxh.cn/73777.Doc
m.wlc.kxnxh.cn/17119.Doc
m.wlc.kxnxh.cn/35913.Doc
m.wlc.kxnxh.cn/91399.Doc
m.wlc.kxnxh.cn/06846.Doc
m.wlc.kxnxh.cn/80268.Doc
m.wlc.kxnxh.cn/71951.Doc
m.wlc.kxnxh.cn/71997.Doc
m.wlc.kxnxh.cn/24846.Doc
m.wlc.kxnxh.cn/33373.Doc
m.wlc.kxnxh.cn/55391.Doc
m.wlc.kxnxh.cn/46624.Doc
m.wlc.kxnxh.cn/80608.Doc
m.wlc.kxnxh.cn/46606.Doc
m.wlx.kxnxh.cn/17771.Doc
m.wlx.kxnxh.cn/66408.Doc
m.wlx.kxnxh.cn/15779.Doc
m.wlx.kxnxh.cn/71571.Doc
m.wlx.kxnxh.cn/51533.Doc
m.wlx.kxnxh.cn/13593.Doc
m.wlx.kxnxh.cn/77911.Doc
m.wlx.kxnxh.cn/97393.Doc
m.wlx.kxnxh.cn/60664.Doc
m.wlx.kxnxh.cn/66008.Doc
m.wlx.kxnxh.cn/39117.Doc
m.wlx.kxnxh.cn/79773.Doc
m.wlx.kxnxh.cn/97319.Doc
m.wlx.kxnxh.cn/46864.Doc
m.wlx.kxnxh.cn/53177.Doc
m.wlx.kxnxh.cn/33731.Doc
m.wlx.kxnxh.cn/46406.Doc
m.wlx.kxnxh.cn/53139.Doc
m.wlx.kxnxh.cn/20062.Doc
m.wlx.kxnxh.cn/15757.Doc
m.wlz.kxnxh.cn/68028.Doc
m.wlz.kxnxh.cn/75513.Doc
m.wlz.kxnxh.cn/04044.Doc
m.wlz.kxnxh.cn/24468.Doc
m.wlz.kxnxh.cn/68204.Doc
m.wlz.kxnxh.cn/66884.Doc
m.wlz.kxnxh.cn/82280.Doc
m.wlz.kxnxh.cn/13171.Doc
m.wlz.kxnxh.cn/17195.Doc
m.wlz.kxnxh.cn/44424.Doc
m.wlz.kxnxh.cn/42688.Doc
m.wlz.kxnxh.cn/28624.Doc
m.wlz.kxnxh.cn/42666.Doc
m.wlz.kxnxh.cn/62060.Doc
m.wlz.kxnxh.cn/62608.Doc
m.wlz.kxnxh.cn/66680.Doc
m.wlz.kxnxh.cn/31135.Doc
m.wlz.kxnxh.cn/91337.Doc
m.wlz.kxnxh.cn/93553.Doc
m.wlz.kxnxh.cn/04420.Doc
m.wll.kxnxh.cn/00026.Doc
m.wll.kxnxh.cn/02884.Doc
m.wll.kxnxh.cn/22288.Doc
m.wll.kxnxh.cn/02020.Doc
m.wll.kxnxh.cn/66066.Doc
m.wll.kxnxh.cn/88880.Doc
m.wll.kxnxh.cn/88668.Doc
m.wll.kxnxh.cn/35993.Doc
m.wll.kxnxh.cn/79151.Doc
m.wll.kxnxh.cn/08822.Doc
m.wll.kxnxh.cn/68820.Doc
m.wll.kxnxh.cn/60440.Doc
m.wll.kxnxh.cn/37799.Doc
m.wll.kxnxh.cn/77519.Doc
m.wll.kxnxh.cn/28626.Doc
m.wll.kxnxh.cn/91335.Doc
m.wll.kxnxh.cn/04848.Doc
m.wll.kxnxh.cn/97571.Doc
m.wll.kxnxh.cn/42280.Doc
m.wll.kxnxh.cn/20680.Doc
m.wlk.kxnxh.cn/68888.Doc
m.wlk.kxnxh.cn/73151.Doc
m.wlk.kxnxh.cn/66040.Doc
m.wlk.kxnxh.cn/88248.Doc
m.wlk.kxnxh.cn/99777.Doc
m.wlk.kxnxh.cn/84202.Doc
m.wlk.kxnxh.cn/91915.Doc
m.wlk.kxnxh.cn/68460.Doc
m.wlk.kxnxh.cn/33511.Doc
m.wlk.kxnxh.cn/99771.Doc
m.wlk.kxnxh.cn/24442.Doc
m.wlk.kxnxh.cn/59771.Doc
m.wlk.kxnxh.cn/99791.Doc
m.wlk.kxnxh.cn/37757.Doc
m.wlk.kxnxh.cn/02846.Doc
m.wlk.kxnxh.cn/06264.Doc
m.wlk.kxnxh.cn/80440.Doc
m.wlk.kxnxh.cn/71577.Doc
m.wlk.kxnxh.cn/95915.Doc
m.wlk.kxnxh.cn/39737.Doc
m.wlj.kxnxh.cn/37571.Doc
m.wlj.kxnxh.cn/51751.Doc
m.wlj.kxnxh.cn/42442.Doc
m.wlj.kxnxh.cn/22006.Doc
m.wlj.kxnxh.cn/46620.Doc
m.wlj.kxnxh.cn/15713.Doc
m.wlj.kxnxh.cn/02066.Doc
m.wlj.kxnxh.cn/53197.Doc
m.wlj.kxnxh.cn/71337.Doc
m.wlj.kxnxh.cn/19973.Doc
m.wlj.kxnxh.cn/17557.Doc
m.wlj.kxnxh.cn/39551.Doc
m.wlj.kxnxh.cn/91779.Doc
m.wlj.kxnxh.cn/77919.Doc
m.wlj.kxnxh.cn/91159.Doc
m.wlj.kxnxh.cn/11755.Doc
m.wlj.kxnxh.cn/13997.Doc
m.wlj.kxnxh.cn/35539.Doc
m.wlj.kxnxh.cn/60808.Doc
m.wlj.kxnxh.cn/28248.Doc
m.wlh.kxnxh.cn/59739.Doc
m.wlh.kxnxh.cn/79313.Doc
m.wlh.kxnxh.cn/64260.Doc
m.wlh.kxnxh.cn/79537.Doc
m.wlh.kxnxh.cn/28048.Doc
m.wlh.kxnxh.cn/86444.Doc
m.wlh.kxnxh.cn/80468.Doc
m.wlh.kxnxh.cn/75953.Doc
m.wlh.kxnxh.cn/97793.Doc
m.wlh.kxnxh.cn/46286.Doc
m.wlh.kxnxh.cn/40260.Doc
m.wlh.kxnxh.cn/31137.Doc
m.wlh.kxnxh.cn/26626.Doc
m.wlh.kxnxh.cn/04422.Doc
m.wlh.kxnxh.cn/57159.Doc
m.wlh.kxnxh.cn/60220.Doc
m.wlh.kxnxh.cn/42206.Doc
m.wlh.kxnxh.cn/08688.Doc
m.wlh.kxnxh.cn/35517.Doc
m.wlh.kxnxh.cn/28660.Doc
m.wlg.kxnxh.cn/24002.Doc
m.wlg.kxnxh.cn/57939.Doc
m.wlg.kxnxh.cn/08624.Doc
m.wlg.kxnxh.cn/68846.Doc
m.wlg.kxnxh.cn/95917.Doc
m.wlg.kxnxh.cn/17773.Doc
m.wlg.kxnxh.cn/97775.Doc
m.wlg.kxnxh.cn/80080.Doc
m.wlg.kxnxh.cn/75935.Doc
m.wlg.kxnxh.cn/13113.Doc
m.wlg.kxnxh.cn/22468.Doc
m.wlg.kxnxh.cn/97357.Doc
m.wlg.kxnxh.cn/02422.Doc
m.wlg.kxnxh.cn/39797.Doc
m.wlg.kxnxh.cn/35995.Doc
m.wlg.kxnxh.cn/33959.Doc
m.wlg.kxnxh.cn/28820.Doc
m.wlg.kxnxh.cn/35797.Doc
m.wlg.kxnxh.cn/80464.Doc
m.wlg.kxnxh.cn/88480.Doc
m.wlf.kxnxh.cn/57133.Doc
m.wlf.kxnxh.cn/95951.Doc
m.wlf.kxnxh.cn/04088.Doc
m.wlf.kxnxh.cn/93359.Doc
m.wlf.kxnxh.cn/00202.Doc
m.wlf.kxnxh.cn/53915.Doc
m.wlf.kxnxh.cn/20286.Doc
m.wlf.kxnxh.cn/62848.Doc
m.wlf.kxnxh.cn/20826.Doc
m.wlf.kxnxh.cn/95957.Doc
m.wlf.kxnxh.cn/00000.Doc
m.wlf.kxnxh.cn/73911.Doc
m.wlf.kxnxh.cn/80648.Doc
m.wlf.kxnxh.cn/33515.Doc
m.wlf.kxnxh.cn/39773.Doc
m.wlf.kxnxh.cn/99995.Doc
m.wlf.kxnxh.cn/57731.Doc
m.wlf.kxnxh.cn/57133.Doc
m.wlf.kxnxh.cn/19339.Doc
m.wlf.kxnxh.cn/04086.Doc
m.wld.kxnxh.cn/39117.Doc
m.wld.kxnxh.cn/37133.Doc
m.wld.kxnxh.cn/48820.Doc
m.wld.kxnxh.cn/77315.Doc
m.wld.kxnxh.cn/17519.Doc
m.wld.kxnxh.cn/33957.Doc
m.wld.kxnxh.cn/44266.Doc
m.wld.kxnxh.cn/11199.Doc
m.wld.kxnxh.cn/08086.Doc
m.wld.kxnxh.cn/13513.Doc
m.wld.kxnxh.cn/00202.Doc
m.wld.kxnxh.cn/19953.Doc
m.wld.kxnxh.cn/04646.Doc
m.wld.kxnxh.cn/24624.Doc
m.wld.kxnxh.cn/51977.Doc
m.wld.kxnxh.cn/17511.Doc
m.wld.kxnxh.cn/57993.Doc
m.wld.kxnxh.cn/97959.Doc
m.wld.kxnxh.cn/79193.Doc
m.wld.kxnxh.cn/77931.Doc
m.wls.kxnxh.cn/66084.Doc
m.wls.kxnxh.cn/00080.Doc
m.wls.kxnxh.cn/06888.Doc
m.wls.kxnxh.cn/84886.Doc
m.wls.kxnxh.cn/15333.Doc
m.wls.kxnxh.cn/33935.Doc
m.wls.kxnxh.cn/64460.Doc
m.wls.kxnxh.cn/62080.Doc
m.wls.kxnxh.cn/26280.Doc
m.wls.kxnxh.cn/68068.Doc
m.wls.kxnxh.cn/51771.Doc
m.wls.kxnxh.cn/35111.Doc
m.wls.kxnxh.cn/42624.Doc
m.wls.kxnxh.cn/75133.Doc
m.wls.kxnxh.cn/39973.Doc
m.wls.kxnxh.cn/59935.Doc
m.wls.kxnxh.cn/59931.Doc
m.wls.kxnxh.cn/91559.Doc
m.wls.kxnxh.cn/06228.Doc
m.wls.kxnxh.cn/26000.Doc
