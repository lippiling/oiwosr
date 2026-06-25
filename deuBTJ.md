
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

qax.wwwcao1314c.cn/84688.Doc
qax.wwwcao1314c.cn/66640.Doc
qax.wwwcao1314c.cn/71571.Doc
qax.wwwcao1314c.cn/46028.Doc
qax.wwwcao1314c.cn/46444.Doc
qax.wwwcao1314c.cn/20440.Doc
qax.wwwcao1314c.cn/26666.Doc
qax.wwwcao1314c.cn/48282.Doc
qax.wwwcao1314c.cn/84064.Doc
qax.wwwcao1314c.cn/79153.Doc
qaz.wwwcao1314c.cn/08442.Doc
qaz.wwwcao1314c.cn/48026.Doc
qaz.wwwcao1314c.cn/95591.Doc
qaz.wwwcao1314c.cn/48602.Doc
qaz.wwwcao1314c.cn/88848.Doc
qaz.wwwcao1314c.cn/80824.Doc
qaz.wwwcao1314c.cn/24440.Doc
qaz.wwwcao1314c.cn/82464.Doc
qaz.wwwcao1314c.cn/46806.Doc
qaz.wwwcao1314c.cn/80864.Doc
qal.wwwcao1314c.cn/00806.Doc
qal.wwwcao1314c.cn/88648.Doc
qal.wwwcao1314c.cn/44400.Doc
qal.wwwcao1314c.cn/02640.Doc
qal.wwwcao1314c.cn/24200.Doc
qal.wwwcao1314c.cn/75195.Doc
qal.wwwcao1314c.cn/60008.Doc
qal.wwwcao1314c.cn/80268.Doc
qal.wwwcao1314c.cn/88408.Doc
qal.wwwcao1314c.cn/84482.Doc
qak.wwwcao1314c.cn/44004.Doc
qak.wwwcao1314c.cn/77357.Doc
qak.wwwcao1314c.cn/62642.Doc
qak.wwwcao1314c.cn/40468.Doc
qak.wwwcao1314c.cn/20248.Doc
qak.wwwcao1314c.cn/86048.Doc
qak.wwwcao1314c.cn/44886.Doc
qak.wwwcao1314c.cn/40826.Doc
qak.wwwcao1314c.cn/17755.Doc
qak.wwwcao1314c.cn/00006.Doc
qaj.wwwcao1314c.cn/20468.Doc
qaj.wwwcao1314c.cn/46848.Doc
qaj.wwwcao1314c.cn/62682.Doc
qaj.wwwcao1314c.cn/60442.Doc
qaj.wwwcao1314c.cn/08680.Doc
qaj.wwwcao1314c.cn/28662.Doc
qaj.wwwcao1314c.cn/66622.Doc
qaj.wwwcao1314c.cn/28626.Doc
qaj.wwwcao1314c.cn/97711.Doc
qaj.wwwcao1314c.cn/64280.Doc
qah.wwwcao1314c.cn/04428.Doc
qah.wwwcao1314c.cn/53175.Doc
qah.wwwcao1314c.cn/62404.Doc
qah.wwwcao1314c.cn/04226.Doc
qah.wwwcao1314c.cn/48868.Doc
qah.wwwcao1314c.cn/46648.Doc
qah.wwwcao1314c.cn/68866.Doc
qah.wwwcao1314c.cn/68662.Doc
qah.wwwcao1314c.cn/62066.Doc
qah.wwwcao1314c.cn/62808.Doc
qag.wwwcao1314c.cn/08684.Doc
qag.wwwcao1314c.cn/20022.Doc
qag.wwwcao1314c.cn/44220.Doc
qag.wwwcao1314c.cn/64462.Doc
qag.wwwcao1314c.cn/84444.Doc
qag.wwwcao1314c.cn/08286.Doc
qag.wwwcao1314c.cn/64448.Doc
qag.wwwcao1314c.cn/28406.Doc
qag.wwwcao1314c.cn/42446.Doc
qag.wwwcao1314c.cn/82828.Doc
qaf.wwwcao1314c.cn/42222.Doc
qaf.wwwcao1314c.cn/44086.Doc
qaf.wwwcao1314c.cn/48486.Doc
qaf.wwwcao1314c.cn/22088.Doc
qaf.wwwcao1314c.cn/62846.Doc
qaf.wwwcao1314c.cn/04868.Doc
qaf.wwwcao1314c.cn/44884.Doc
qaf.wwwcao1314c.cn/42848.Doc
qaf.wwwcao1314c.cn/95353.Doc
qaf.wwwcao1314c.cn/20624.Doc
qad.wwwcao1314c.cn/02468.Doc
qad.wwwcao1314c.cn/80460.Doc
qad.wwwcao1314c.cn/86606.Doc
qad.wwwcao1314c.cn/06426.Doc
qad.wwwcao1314c.cn/08262.Doc
qad.wwwcao1314c.cn/13515.Doc
qad.wwwcao1314c.cn/17711.Doc
qad.wwwcao1314c.cn/88688.Doc
qad.wwwcao1314c.cn/44222.Doc
qad.wwwcao1314c.cn/22064.Doc
qas.wwwcao1314c.cn/86084.Doc
qas.wwwcao1314c.cn/84600.Doc
qas.wwwcao1314c.cn/44422.Doc
qas.wwwcao1314c.cn/48248.Doc
qas.wwwcao1314c.cn/48820.Doc
qas.wwwcao1314c.cn/62224.Doc
qas.wwwcao1314c.cn/48440.Doc
qas.wwwcao1314c.cn/04808.Doc
qas.wwwcao1314c.cn/86068.Doc
qas.wwwcao1314c.cn/86824.Doc
qaa.wwwcao1314c.cn/68040.Doc
qaa.wwwcao1314c.cn/68268.Doc
qaa.wwwcao1314c.cn/04042.Doc
qaa.wwwcao1314c.cn/26046.Doc
qaa.wwwcao1314c.cn/88020.Doc
qaa.wwwcao1314c.cn/73317.Doc
qaa.wwwcao1314c.cn/24484.Doc
qaa.wwwcao1314c.cn/82620.Doc
qaa.wwwcao1314c.cn/08240.Doc
qaa.wwwcao1314c.cn/26420.Doc
qap.wwwcao1314c.cn/60888.Doc
qap.wwwcao1314c.cn/17739.Doc
qap.wwwcao1314c.cn/88022.Doc
qap.wwwcao1314c.cn/48606.Doc
qap.wwwcao1314c.cn/04204.Doc
qap.wwwcao1314c.cn/28644.Doc
qap.wwwcao1314c.cn/04466.Doc
qap.wwwcao1314c.cn/73939.Doc
qap.wwwcao1314c.cn/84882.Doc
qap.wwwcao1314c.cn/60840.Doc
qao.wwwcao1314c.cn/46806.Doc
qao.wwwcao1314c.cn/53397.Doc
qao.wwwcao1314c.cn/20028.Doc
qao.wwwcao1314c.cn/26646.Doc
qao.wwwcao1314c.cn/46628.Doc
qao.wwwcao1314c.cn/84600.Doc
qao.wwwcao1314c.cn/20880.Doc
qao.wwwcao1314c.cn/17395.Doc
qao.wwwcao1314c.cn/24288.Doc
qao.wwwcao1314c.cn/46400.Doc
qai.wwwcao1314c.cn/24228.Doc
qai.wwwcao1314c.cn/66828.Doc
qai.wwwcao1314c.cn/02844.Doc
qai.wwwcao1314c.cn/68822.Doc
qai.wwwcao1314c.cn/60626.Doc
qai.wwwcao1314c.cn/48444.Doc
qai.wwwcao1314c.cn/68604.Doc
qai.wwwcao1314c.cn/06644.Doc
qai.wwwcao1314c.cn/64000.Doc
qai.wwwcao1314c.cn/00084.Doc
qau.wwwcao1314c.cn/80082.Doc
qau.wwwcao1314c.cn/20042.Doc
qau.wwwcao1314c.cn/82408.Doc
qau.wwwcao1314c.cn/84464.Doc
qau.wwwcao1314c.cn/44628.Doc
qau.wwwcao1314c.cn/48668.Doc
qau.wwwcao1314c.cn/88464.Doc
qau.wwwcao1314c.cn/80620.Doc
qau.wwwcao1314c.cn/86602.Doc
qau.wwwcao1314c.cn/42448.Doc
qay.wwwcao1314c.cn/82642.Doc
qay.wwwcao1314c.cn/28064.Doc
qay.wwwcao1314c.cn/08442.Doc
qay.wwwcao1314c.cn/79917.Doc
qay.wwwcao1314c.cn/24062.Doc
qay.wwwcao1314c.cn/08866.Doc
qay.wwwcao1314c.cn/79131.Doc
qay.wwwcao1314c.cn/64260.Doc
qay.wwwcao1314c.cn/80004.Doc
qay.wwwcao1314c.cn/48860.Doc
qat.wwwcao1314c.cn/64046.Doc
qat.wwwcao1314c.cn/44284.Doc
qat.wwwcao1314c.cn/40808.Doc
qat.wwwcao1314c.cn/20620.Doc
qat.wwwcao1314c.cn/44026.Doc
qat.wwwcao1314c.cn/44208.Doc
qat.wwwcao1314c.cn/06040.Doc
qat.wwwcao1314c.cn/33173.Doc
qat.wwwcao1314c.cn/20484.Doc
qat.wwwcao1314c.cn/84860.Doc
qar.wwwcao1314c.cn/06248.Doc
qar.wwwcao1314c.cn/00642.Doc
qar.wwwcao1314c.cn/40880.Doc
qar.wwwcao1314c.cn/64802.Doc
qar.wwwcao1314c.cn/60668.Doc
qar.wwwcao1314c.cn/22062.Doc
qar.wwwcao1314c.cn/66420.Doc
qar.wwwcao1314c.cn/02486.Doc
qar.wwwcao1314c.cn/84642.Doc
qar.wwwcao1314c.cn/48840.Doc
qae.wwwcao1314c.cn/84622.Doc
qae.wwwcao1314c.cn/60842.Doc
qae.wwwcao1314c.cn/28040.Doc
qae.wwwcao1314c.cn/20088.Doc
qae.wwwcao1314c.cn/82286.Doc
qae.wwwcao1314c.cn/62824.Doc
qae.wwwcao1314c.cn/64800.Doc
qae.wwwcao1314c.cn/68086.Doc
qae.wwwcao1314c.cn/42420.Doc
qae.wwwcao1314c.cn/42824.Doc
qaw.wwwcao1314c.cn/88666.Doc
qaw.wwwcao1314c.cn/24086.Doc
qaw.wwwcao1314c.cn/60408.Doc
qaw.wwwcao1314c.cn/22682.Doc
qaw.wwwcao1314c.cn/24088.Doc
qaw.wwwcao1314c.cn/37991.Doc
qaw.wwwcao1314c.cn/46620.Doc
qaw.wwwcao1314c.cn/26442.Doc
qaw.wwwcao1314c.cn/22008.Doc
qaw.wwwcao1314c.cn/26684.Doc
qaq.wwwcao1314c.cn/22684.Doc
qaq.wwwcao1314c.cn/64666.Doc
qaq.wwwcao1314c.cn/86084.Doc
qaq.wwwcao1314c.cn/44240.Doc
qaq.wwwcao1314c.cn/62006.Doc
qaq.wwwcao1314c.cn/22604.Doc
qaq.wwwcao1314c.cn/08822.Doc
qaq.wwwcao1314c.cn/04022.Doc
qaq.wwwcao1314c.cn/02682.Doc
qaq.wwwcao1314c.cn/59931.Doc
qpm.wwwcao1314c.cn/82084.Doc
qpm.wwwcao1314c.cn/86444.Doc
qpm.wwwcao1314c.cn/82044.Doc
qpm.wwwcao1314c.cn/62864.Doc
qpm.wwwcao1314c.cn/57951.Doc
qpm.wwwcao1314c.cn/71511.Doc
qpm.wwwcao1314c.cn/48668.Doc
qpm.wwwcao1314c.cn/20842.Doc
qpm.wwwcao1314c.cn/68260.Doc
qpm.wwwcao1314c.cn/80646.Doc
qpn.wwwcao1314c.cn/44048.Doc
qpn.wwwcao1314c.cn/28408.Doc
qpn.wwwcao1314c.cn/86240.Doc
qpn.wwwcao1314c.cn/42488.Doc
qpn.wwwcao1314c.cn/48284.Doc
qpn.wwwcao1314c.cn/02420.Doc
qpn.wwwcao1314c.cn/44848.Doc
qpn.wwwcao1314c.cn/04268.Doc
qpn.wwwcao1314c.cn/68468.Doc
qpn.wwwcao1314c.cn/48246.Doc
qpb.wwwcao1314c.cn/44884.Doc
qpb.wwwcao1314c.cn/20806.Doc
qpb.wwwcao1314c.cn/88826.Doc
qpb.wwwcao1314c.cn/60884.Doc
qpb.wwwcao1314c.cn/48260.Doc
qpb.wwwcao1314c.cn/00602.Doc
qpb.wwwcao1314c.cn/64200.Doc
qpb.wwwcao1314c.cn/88688.Doc
qpb.wwwcao1314c.cn/40220.Doc
qpb.wwwcao1314c.cn/80464.Doc
qpv.wwwcao1314c.cn/60884.Doc
qpv.wwwcao1314c.cn/44026.Doc
qpv.wwwcao1314c.cn/68264.Doc
qpv.wwwcao1314c.cn/88626.Doc
qpv.wwwcao1314c.cn/86604.Doc
qpv.wwwcao1314c.cn/88424.Doc
qpv.wwwcao1314c.cn/62860.Doc
qpv.wwwcao1314c.cn/80622.Doc
qpv.wwwcao1314c.cn/06806.Doc
qpv.wwwcao1314c.cn/02482.Doc
qpc.wwwcao1314c.cn/66084.Doc
qpc.wwwcao1314c.cn/28062.Doc
qpc.wwwcao1314c.cn/88802.Doc
qpc.wwwcao1314c.cn/44204.Doc
qpc.wwwcao1314c.cn/80400.Doc
qpc.wwwcao1314c.cn/24626.Doc
qpc.wwwcao1314c.cn/44280.Doc
qpc.wwwcao1314c.cn/62804.Doc
qpc.wwwcao1314c.cn/08846.Doc
qpc.wwwcao1314c.cn/02402.Doc
qpx.wwwcao1314c.cn/48288.Doc
qpx.wwwcao1314c.cn/66620.Doc
qpx.wwwcao1314c.cn/00286.Doc
qpx.wwwcao1314c.cn/22226.Doc
qpx.wwwcao1314c.cn/04224.Doc
qpx.wwwcao1314c.cn/04880.Doc
qpx.wwwcao1314c.cn/60084.Doc
qpx.wwwcao1314c.cn/60206.Doc
qpx.wwwcao1314c.cn/53717.Doc
qpx.wwwcao1314c.cn/86020.Doc
qpz.wwwcao1314c.cn/48082.Doc
qpz.wwwcao1314c.cn/24248.Doc
qpz.wwwcao1314c.cn/20002.Doc
qpz.wwwcao1314c.cn/06662.Doc
qpz.wwwcao1314c.cn/48846.Doc
qpz.wwwcao1314c.cn/82082.Doc
qpz.wwwcao1314c.cn/02440.Doc
qpz.wwwcao1314c.cn/84828.Doc
qpz.wwwcao1314c.cn/44446.Doc
qpz.wwwcao1314c.cn/22822.Doc
qpl.wwwcao1314c.cn/40624.Doc
qpl.wwwcao1314c.cn/64442.Doc
qpl.wwwcao1314c.cn/84264.Doc
qpl.wwwcao1314c.cn/04020.Doc
qpl.wwwcao1314c.cn/24604.Doc
qpl.wwwcao1314c.cn/46826.Doc
qpl.wwwcao1314c.cn/62200.Doc
qpl.wwwcao1314c.cn/26684.Doc
qpl.wwwcao1314c.cn/44282.Doc
qpl.wwwcao1314c.cn/22882.Doc
qpk.wwwcao1314c.cn/40048.Doc
qpk.wwwcao1314c.cn/26806.Doc
qpk.wwwcao1314c.cn/48020.Doc
qpk.wwwcao1314c.cn/84600.Doc
qpk.wwwcao1314c.cn/22220.Doc
qpk.wwwcao1314c.cn/46682.Doc
qpk.wwwcao1314c.cn/60006.Doc
qpk.wwwcao1314c.cn/42242.Doc
qpk.wwwcao1314c.cn/84044.Doc
qpk.wwwcao1314c.cn/62822.Doc
qpj.wwwcao1314c.cn/84280.Doc
qpj.wwwcao1314c.cn/84468.Doc
qpj.wwwcao1314c.cn/86264.Doc
qpj.wwwcao1314c.cn/80480.Doc
qpj.wwwcao1314c.cn/44606.Doc
qpj.wwwcao1314c.cn/64444.Doc
qpj.wwwcao1314c.cn/46040.Doc
qpj.wwwcao1314c.cn/26646.Doc
qpj.wwwcao1314c.cn/44040.Doc
qpj.wwwcao1314c.cn/24868.Doc
qph.wwwcao1314c.cn/08266.Doc
qph.wwwcao1314c.cn/26042.Doc
qph.wwwcao1314c.cn/42062.Doc
qph.wwwcao1314c.cn/02062.Doc
qph.wwwcao1314c.cn/26220.Doc
qph.wwwcao1314c.cn/88442.Doc
qph.wwwcao1314c.cn/84440.Doc
qph.wwwcao1314c.cn/00446.Doc
qph.wwwcao1314c.cn/42822.Doc
qph.wwwcao1314c.cn/46022.Doc
qpg.wwwcao1314c.cn/82486.Doc
qpg.wwwcao1314c.cn/24400.Doc
qpg.wwwcao1314c.cn/40228.Doc
qpg.wwwcao1314c.cn/40486.Doc
qpg.wwwcao1314c.cn/20602.Doc
qpg.wwwcao1314c.cn/08264.Doc
qpg.wwwcao1314c.cn/20262.Doc
qpg.wwwcao1314c.cn/20080.Doc
qpg.wwwcao1314c.cn/88442.Doc
qpg.wwwcao1314c.cn/82048.Doc
qpf.wwwcao1314c.cn/80440.Doc
qpf.wwwcao1314c.cn/42826.Doc
qpf.wwwcao1314c.cn/26826.Doc
qpf.wwwcao1314c.cn/66840.Doc
qpf.wwwcao1314c.cn/62600.Doc
qpf.wwwcao1314c.cn/84264.Doc
qpf.wwwcao1314c.cn/22280.Doc
qpf.wwwcao1314c.cn/22662.Doc
qpf.wwwcao1314c.cn/22448.Doc
qpf.wwwcao1314c.cn/64602.Doc
qpd.wwwcao1314c.cn/08868.Doc
qpd.wwwcao1314c.cn/42080.Doc
qpd.wwwcao1314c.cn/71599.Doc
qpd.wwwcao1314c.cn/40626.Doc
qpd.wwwcao1314c.cn/06226.Doc
qpd.wwwcao1314c.cn/82460.Doc
qpd.wwwcao1314c.cn/44028.Doc
qpd.wwwcao1314c.cn/15577.Doc
qpd.wwwcao1314c.cn/40026.Doc
qpd.wwwcao1314c.cn/22020.Doc
qps.wwwcao1314c.cn/68642.Doc
qps.wwwcao1314c.cn/00644.Doc
qps.wwwcao1314c.cn/80620.Doc
qps.wwwcao1314c.cn/24406.Doc
qps.wwwcao1314c.cn/95973.Doc
qps.wwwcao1314c.cn/53315.Doc
qps.wwwcao1314c.cn/28226.Doc
qps.wwwcao1314c.cn/04808.Doc
qps.wwwcao1314c.cn/84862.Doc
qps.wwwcao1314c.cn/42820.Doc
qpa.wwwcao1314c.cn/08000.Doc
qpa.wwwcao1314c.cn/84246.Doc
qpa.wwwcao1314c.cn/00602.Doc
qpa.wwwcao1314c.cn/20682.Doc
qpa.wwwcao1314c.cn/86802.Doc
qpa.wwwcao1314c.cn/64884.Doc
qpa.wwwcao1314c.cn/62062.Doc
qpa.wwwcao1314c.cn/02655.Doc
qpa.wwwcao1314c.cn/95939.Doc
qpa.wwwcao1314c.cn/48600.Doc
qpp.wwwcao1314c.cn/28686.Doc
qpp.wwwcao1314c.cn/20084.Doc
qpp.wwwcao1314c.cn/02684.Doc
qpp.wwwcao1314c.cn/64664.Doc
qpp.wwwcao1314c.cn/48282.Doc
qpp.wwwcao1314c.cn/66886.Doc
qpp.wwwcao1314c.cn/44080.Doc
qpp.wwwcao1314c.cn/26406.Doc
qpp.wwwcao1314c.cn/26688.Doc
qpp.wwwcao1314c.cn/64462.Doc
qpo.wwwcao1314c.cn/19977.Doc
qpo.wwwcao1314c.cn/22608.Doc
qpo.wwwcao1314c.cn/44406.Doc
qpo.wwwcao1314c.cn/75371.Doc
qpo.wwwcao1314c.cn/28800.Doc
qpo.wwwcao1314c.cn/86600.Doc
qpo.wwwcao1314c.cn/48848.Doc
qpo.wwwcao1314c.cn/20400.Doc
qpo.wwwcao1314c.cn/00622.Doc
qpo.wwwcao1314c.cn/88628.Doc
qpi.wwwcao1314c.cn/02242.Doc
qpi.wwwcao1314c.cn/40688.Doc
qpi.wwwcao1314c.cn/80640.Doc
qpi.wwwcao1314c.cn/28422.Doc
qpi.wwwcao1314c.cn/95517.Doc
qpi.wwwcao1314c.cn/24866.Doc
qpi.wwwcao1314c.cn/44802.Doc
qpi.wwwcao1314c.cn/68662.Doc
qpi.wwwcao1314c.cn/04668.Doc
qpi.wwwcao1314c.cn/28602.Doc
qpu.wwwcao1314c.cn/86224.Doc
qpu.wwwcao1314c.cn/02088.Doc
qpu.wwwcao1314c.cn/02860.Doc
qpu.wwwcao1314c.cn/84048.Doc
qpu.wwwcao1314c.cn/02604.Doc
qpu.wwwcao1314c.cn/66800.Doc
qpu.wwwcao1314c.cn/28444.Doc
qpu.wwwcao1314c.cn/11735.Doc
qpu.wwwcao1314c.cn/06604.Doc
qpu.wwwcao1314c.cn/84480.Doc
qpy.wwwcao1314c.cn/88066.Doc
qpy.wwwcao1314c.cn/46220.Doc
qpy.wwwcao1314c.cn/46426.Doc
qpy.wwwcao1314c.cn/82288.Doc
qpy.wwwcao1314c.cn/82286.Doc
qpy.wwwcao1314c.cn/97115.Doc
qpy.wwwcao1314c.cn/42284.Doc
qpy.wwwcao1314c.cn/82468.Doc
qpy.wwwcao1314c.cn/28840.Doc
qpy.wwwcao1314c.cn/68066.Doc
qpt.wwwcao1314c.cn/48606.Doc
qpt.wwwcao1314c.cn/44242.Doc
qpt.wwwcao1314c.cn/04082.Doc
qpt.wwwcao1314c.cn/00442.Doc
qpt.wwwcao1314c.cn/62448.Doc
qpt.wwwcao1314c.cn/02840.Doc
qpt.wwwcao1314c.cn/20022.Doc
qpt.wwwcao1314c.cn/84208.Doc
qpt.wwwcao1314c.cn/44266.Doc
qpt.wwwcao1314c.cn/82642.Doc
qpr.wwwcao1314c.cn/20606.Doc
qpr.wwwcao1314c.cn/08222.Doc
qpr.wwwcao1314c.cn/29518.Doc
qpr.wwwcao1314c.cn/77882.Doc
qpr.wwwcao1314c.cn/95092.Doc
qpr.wwwcao1314c.cn/72174.Doc
qpr.wwwcao1314c.cn/14693.Doc
qpr.wwwcao1314c.cn/42975.Doc
qpr.wwwcao1314c.cn/67550.Doc
qpr.wwwcao1314c.cn/94605.Doc
qpe.wwwcao1314c.cn/16594.Doc
qpe.wwwcao1314c.cn/03929.Doc
qpe.wwwcao1314c.cn/30620.Doc
qpe.wwwcao1314c.cn/87074.Doc
qpe.wwwcao1314c.cn/18977.Doc
qpe.wwwcao1314c.cn/66649.Doc
qpe.wwwcao1314c.cn/20970.Doc
qpe.wwwcao1314c.cn/22620.Doc
qpe.wwwcao1314c.cn/28853.Doc
qpe.wwwcao1314c.cn/89187.Doc
qpw.wwwcao1314c.cn/61276.Doc
qpw.wwwcao1314c.cn/12440.Doc
qpw.wwwcao1314c.cn/46461.Doc
qpw.wwwcao1314c.cn/92930.Doc
qpw.wwwcao1314c.cn/40933.Doc
qpw.wwwcao1314c.cn/95603.Doc
qpw.wwwcao1314c.cn/86608.Doc
qpw.wwwcao1314c.cn/84604.Doc
qpw.wwwcao1314c.cn/86628.Doc
qpw.wwwcao1314c.cn/60488.Doc
qpq.wwwcao1314c.cn/04482.Doc
qpq.wwwcao1314c.cn/86246.Doc
qpq.wwwcao1314c.cn/04844.Doc
qpq.wwwcao1314c.cn/86002.Doc
qpq.wwwcao1314c.cn/04282.Doc
qpq.wwwcao1314c.cn/24602.Doc
qpq.wwwcao1314c.cn/22246.Doc
qpq.wwwcao1314c.cn/20468.Doc
qpq.wwwcao1314c.cn/24884.Doc
qpq.wwwcao1314c.cn/86602.Doc
qom.wwwcao1314c.cn/42884.Doc
qom.wwwcao1314c.cn/40440.Doc
qom.wwwcao1314c.cn/88624.Doc
qom.wwwcao1314c.cn/62280.Doc
qom.wwwcao1314c.cn/62664.Doc
qom.wwwcao1314c.cn/79993.Doc
qom.wwwcao1314c.cn/48420.Doc
qom.wwwcao1314c.cn/60048.Doc
qom.wwwcao1314c.cn/24284.Doc
qom.wwwcao1314c.cn/02020.Doc
qon.wwwcao1314c.cn/80284.Doc
qon.wwwcao1314c.cn/46880.Doc
qon.wwwcao1314c.cn/82860.Doc
qon.wwwcao1314c.cn/84024.Doc
qon.wwwcao1314c.cn/08460.Doc
qon.wwwcao1314c.cn/42282.Doc
qon.wwwcao1314c.cn/06028.Doc
qon.wwwcao1314c.cn/84488.Doc
qon.wwwcao1314c.cn/37391.Doc
qon.wwwcao1314c.cn/06604.Doc
